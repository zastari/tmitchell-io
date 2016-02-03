---
date: 2016-01-30T23:51:19Z
draft: false
title: Debugging Linux Process Memory Usage
category: Linux
tags:
 - linux
 - memory
 - gdb
 - mysql
 - valgrind
---
I recently encountered a MySQL installation using a larger memory footprint than I expected.  I will explain the troubleshooting process as well as how to effectively use tools such as *free*, *gdb*, *ps*, *pmap*, and *valgrind* to solve similar issues.

# Step 1: Triage

## Identify System Utilization with **free**
{{< highlight bash >}}
$ free -m
             total       used       free     shared    buffers     cached
Mem:           995        883        112          0         12        358
-/+ buffers/cache:        512        483
Swap:         2047         18       2029
{{< /highlight >}}

While *free*'s output is quite simple, it does tell us crucial information:

  - **Swap used**: How much memory has been swapped to disk (18M)
  - **Memory used**: How much physical memory has been mapped by userspace processes (512M)
  - **Cache used**: How much memory has been used by kernel caches such as the <a href="https://www.thomas-krenn.com/en/wiki/Linux_Page_Cache_Basics" title="Linux Page Cache Basics">page cache</a> and <a href="http://www.secretmango.com/jimb/Whitepapers/slabs/slab.html" title="slabs">slabs</a>. (370M)

Overall, this output tells us that we have 512M of non-reclaimable memory allocated.

## Identify Process Utilization with **ps**
{{< highlight bash >}}
$ ps -o command,pid,rss,vsz --sort:rss -e | tail -3
/bin/bash                   17575  2640 110448
/bin/bash                   19212  3752 110448
/usr/sbin/mysqld --basedir= 20476 454384 1129292
{{< /highlight >}}
*ps* provides two useful metrics for memory utilization:

  - **rss**: The Resident Set Size of a process. This measures the amount of physical memory the process is using.
  - **vsz**: The Virtual Size of a process. This measures the amount of memory a process has currently requested.

Because we expect that processes will typically ask for much more memory than they need (see: <a href="https://www.kernel.org/doc/Documentation/vm/overcommit-accounting" title="Overcommit Memory">vm.overcommit_memory</a>), we are only concerned with the RSS of a process.
From this output, we know that MySQL has requested 1,129,292K of memory and is actively using 454,384K.


## Identify Per-Process Allocations with **pmap**
{{< highlight bash >}}
$ sudo pmap -x $(pidof mysqld) | sort -nk3
Address           Kbytes     RSS   Dirty Mode   Mapping
...
0000003606e00000    1764      32       0 r-x--  libcrypto.so.1.0.1e
0000003602200000      92      40       0 r-x--  libpthread-2.12.so
00007fff5a6a6000      84      52      52 rw---    [ stack ]
...
000000000221b000    7604    7396    7396 rw---    [ anon ]
00007f8d6e220000  161664   16744   16744 rw---    [ anon ]
00007f8d7d2ed000  414776  414776  414776 rw---    [ anon ]
total kB         1195088  455216  448316
{{< /highlight >}}
*pmap* allows us to look at the individual allocations a process has made.
The -x flag provides both Virtual and Physical mappings; VSZ is provided under the "Kbytes" column.
The mapping column lists the type of memory at the given address:

  - Shared libraries. In these cases, the name of the shared library is explicitly given in the mapping.
  - The process's stack. This is identified by **[ stack ]**.
  - Heap allocations. These are identified by **[ anon ]**.

This output indicates that almost all of the memory is being used by a heap allocation at address *0x7f8d7d2ed000*.

# Step 2: Find the Cause

Now that we know what sort of memory allocation we are looking at, we have several different ways we can proceed:

  - Use domain-specific knowledge of the application to reason about the source.
  - Figure out how the mapping changes over time.
  - See if we can find the source of the allocation in a debugger.

We will refrain from using domain-specific knowledge.

## Understand the Allocation over Time

From here, we can simply check *pmap* and see if the allocation changes.
In this case, it doesn't.
Now we know several important facts about the nature of this allocation:

  - It is physically using all of the space it has requested.
  - It is not growing in size.
  - It is persistent and not being unallocated.

Given these facts, there is a good chance that this allocation occurs on or near startup.
Our next logical choice is to move traffic away from our MySQL server, restart it, and see if the allocation comes back. It does:

{{< highlight bash >}}
$ sudo pmap -x $(pidof mysqld) | sort -nk3 | tail -2
00007f3a1dc7b000  414776  414776  414776 rw---    [ anon ]
total kB         1129292  453824  447548
{{< /highlight >}}

## Profile the Allocation with Massif

Now that we have a consistent way to recreate the allocation occurring, we can measure it over time using <a href="http://valgrind.org/docs/manual/ms-manual.html" title="Valgrind massif">Valgrind's *massif* tool</a>.


### Execute Massif
{{< highlight bash >}}
pts2$ sudo service mysql stop
pts1$ sudo -u mysql \
      valgrind --tool=massif /usr/sbin/mysqld \
      --basedir=/usr --datadir=/var/lib/mysql ... [other args omitted] ...
pts2$ sudo mysqladmin shutdown # after startup completes
$ sudo -u mysql ms_print massif.out.21223
...
99.86% (87,167,792B) (heap allocation functions) malloc/new/new[], --alloc-fns, etc.
->88.01% (76,824,776B) 0xB15288: pfs_malloc(unsigned long, int) (pfs_global.cc:57)
...
{{< /highlight >}}

The full output of the massif run is available <a href="/static/process-memory-usage/massif_output.txt" title="massif output">here</a>.
This output tells us that 88.01% of allocated heap is coming from the pfs_malloc call in pfs_global.cc.
Looking at the MySQL source tells us that this comes from perfschema:

{{< highlight bash >}}
$ find mysql-5.6.28 -name "pfs_global.cc"
mysql-5.6.28/storage/perfschema/pfs_global.cc
{{< /highlight >}}

Note that this heap allocation is much smaller than the RSS that mysqld occupies.
This is due to the 64 byte alignment given in the posix_memalign calls that pfs_malloc uses to <a href="http://igoro.com/archive/gallery-of-processor-cache-effects/" title="Gallery of Processor Cache Effects">align to x86 cache lines</a>.
(See: <a href="http://stackoverflow.com/questions/13490718/track-memory-usage-in-c-and-evaluate-memory-consumption" title="Page heaps with Massif">Page heap debugging with Massif</a>)

## Confirm the Cause

Now we know that something called *perfschema* is responsible for the memory usage.
A quick review of <a href="http://dev.mysql.com/doc/refman/5.6/en/performance-schema-system-variables.html#sysvar_performance_schema" title="MySQL system variables">MySQL's system variables</a> indicates that performance_schema = 1 is set by default.
Let's see what happens if we turn it off and restart MySQL:

{{< highlight bash >}}
$ ps -orss,vsz $(pidof mysqld)
  RSS    VSZ
41992 713964
$ free -m
             total       used       free     shared    buffers     cached
Mem:           995        677        318          0         14        540
-/+ buffers/cache:        122        873
Swap:         2047         18       2029
{{< /highlight >}}

MySQL is now using 1/10th as much memory and userspace memory is down by almost a factor of 5!
Unfortunately, we are disabling a feature that we may need, so this is not an acceptable solution.
We need a better understanding of why performance_schema is using memory and what we can do to reduce it.

# Step 3: Disassemble the Cause

<a href="/static/process-memory-usage/massif_output.txt" title="massif output">The output from Massif</a> tells us that pfs_malloc is called repeatedly while initializing various instruments.
If we want to know what allocations are being made and what the size of each one is, we can inspect the process on startup with gdb and track each call.
We do this by setting a break on pfs_malloc, running the program, and then inspecting the stack each time it breaks on a pfs_malloc call.

## Run MySQL with gdb
{{< highlight bash >}}
$ sudo -u mysql \
  gdb --args /usr/sbin/mysqld \
  --basedir=/usr ... [extra args omitted] ...
{{< /highlight >}}

Note that to pass additional flags, such as --basedir, gdb must be passed the --args flag to indicate that all subsequent arguments are flags for the program to be executed.

## Set a Breakpoint
{{< highlight c >}}
(gdb) br pfs_malloc
Breakpoint 1 at 0xb15269: file /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_global.cc, line 57.
(gdb) set logging on
Copying output to gdb.txt.
(gdb) run
Starting program: /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib64/mysql/plugin --user=mysql --log-error=/var/log/mysqld.log --pid-file=/var/run/mysqld/mysqld.pid --socket=/var/lib/mysql/mysql.sock
{{< /highlight >}}

Our first task is to halt the program when a call to pfs_malloc is made.
The *breakpoint* command provides this functionality.
From here, we can execute *run* which will execute the program until a breakpoint is hit.

## Analyze the Process at a Breakpoint
{{< highlight c >}}
Breakpoint 1, pfs_malloc (size=51200, flags=32) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_global.cc:57
57        if (unlikely(posix_memalign(& ptr, PFS_ALIGNEMENT, size)))
(gdb) info threads
* 1 Thread 0x7ffff7fe67e0 (LWP 22031)  pfs_malloc (size=51200, flags=32) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_global.cc:57
(gdb) info stack
#0  pfs_malloc (size=51200, flags=32) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_global.cc:57
#1  0x0000000000b1b1c2 in init_sync_class (mutex_class_sizing=Unhandled dwarf expression opcode 0xf3
) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_instr_class.cc:258
#2  0x0000000000b1e966 in initialize_performance_schema (param=0x13556a0) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_server.cc:86
#3  0x000000000059283d in mysqld_main (argc=17, argv=0x135fc58) at /usr/src/debug/percona-server-5.6.28-76.1/sql/mysqld.cc:5617
#4  0x00007ffff6f83d5d in __libc_start_main () from /lib64/libc.so.6
#5  0x0000000000584f1d in _start ()
(gdb) frame 1
#1  0x0000000000b1b1c2 in init_sync_class (mutex_class_sizing=Unhandled dwarf expression opcode 0xf3
) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_instr_class.cc:258
258         mutex_class_array= PFS_MALLOC_ARRAY(mutex_class_max, sizeof(PFS_mutex_class),
{{< /highlight >}}

*gdb* breaks programs down into the following order:

  1. **Inferiors**: Applications being debugged. In this case, the mysqld process.
  2. **Threads**: Process threads. At this point, MySQL is still running under the single main thread.
  3. **Frames**: These are individual calls on the call stack of a thread. They are organized by traversing the frame pointers from the deepest frame to the top of the stack.
  4. **Blocks**: These provide the symbol table for individual frames.

In this example, we see that mysqld is running single threaded.
Frame 0 provides the arguments to the pfs_malloc call.
Here, we can see that a size argument of 51200 bytes is being passed.
Frame 1 of this thread will provide the block that calls pfs_malloc; by stepping into this frame we can find where these allocations originate.
In this case, we see that PFS_MALLOC_ARRAY takes a counter and a size and calls pfs_malloc with this information.

## Measure Each Allocation
{{< highlight c >}}
(gdb) command 1
Type commands for breakpoint(s) 1, one per line.
End with a line saying just "end".
>frame 1
>end
(gdb) continue
Continuing.

Breakpoint 1, pfs_malloc (size=12800, flags=32) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_global.cc:57
57        if (unlikely(posix_memalign(& ptr, PFS_ALIGNEMENT, size)))
#1  0x0000000000b1b19f in init_sync_class (mutex_class_sizing=Unhandled dwarf expression opcode 0xf3
) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_instr_class.cc:266
266         rwlock_class_array= PFS_MALLOC_ARRAY(rwlock_class_max, sizeof(PFS_rwlock_class),
{{< /highlight >}}

We can configure gdb to automatically inspect the first frame each time a breakpoint is hit by using *command*.
Next, we use *continue* to resume execution.

Now we can repeatedly hit enter to continue at each break, or we can use a scripted gdb command or the <a href="https://sourceware.org/gdb/onlinedocs/gdb/Python.html" title="gdb Python">gdb Python library</a> to automate this process. The resulting output file is available <a href="/static/process-memory-usage/gdb_output.txt" title="gdb output">here</a>.

## Tie Everything Together

Now that we have gdb output of each allocation, we can use our favorite string mangling language to parse and sort the size results:
{{< highlight bash >}}
$ grep -o 'size=[0-9]\+' gdb.txt | cut -d'=' -f2 | sort -n | tail -5
5120000
5716480
5898240
9485568
15190272
{{< /highlight >}}

Let's look at the allocation of 15190272 bytes in gdb.txt to see why it's so large:

{{< highlight c >}}
Breakpoint 1, pfs_malloc (size=15190272, flags=32) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_global.cc:57
57    if (unlikely(posix_memalign(& ptr, PFS_ALIGNEMENT, size)))
#1  0x0000000000b1b393 in init_table_share (table_share_sizing=Unhandled dwarf expression opcode 0xf3
) at /usr/src/debug/percona-server-5.6.28-76.1/storage/perfschema/pfs_instr_class.cc:344
344     table_share_array= PFS_MALLOC_ARRAY(table_share_max, sizeof(PFS_table_share),
{{< /highlight >}}

This indicates that the allocation happens in pfs_instr_class.cc around line 344. Let's look at the function that calls this:

{{< highlight c >}}
# storage/perfschema/pfs_instr_class.cc
 336 int init_table_share(uint table_share_sizing)
 337 {
 338   int result= 0;
 339   table_share_max= table_share_sizing;
 340   table_share_lost= 0;
 341
 342   if (table_share_max > 0)
 343   {
 344     table_share_array= PFS_MALLOC_ARRAY(table_share_max, sizeof(PFS_table_share),
 345                                         PFS_table_share, MYF(MY_ZEROFILL));
 346     if (unlikely(table_share_array == NULL))
 347       result= 1;
 348   }
 349   else
 350     table_share_array= NULL;
 351
 352   return result;
 353 }
{{< /highlight >}}

Here we see that PFS_MALLOC_ARRAY allocates table_share_max blocks of size PFS_table_share, and table_share_max is set to table_share_sizing.
Digging through table_share_sizing's history, we find that it is ultimately assigned from pfs_autosize.cc:

{{< highlight c >}}
# storage/perfschema/pfs_autosize.cc
197 static void apply_heuristic(PFS_global_param *p, PFS_sizing_data *h)
198 {
199   ulong count;
200   ulong con = p->m_hints.m_max_connections;
201   ulong handle = p->m_hints.m_table_open_cache;
202   ulong share = p->m_hints.m_table_definition_cache;
203   ulong file = p->m_hints.m_open_files_limit;
...
212   if (p->m_table_share_sizing < 0)
213   {
214     count= share;
215
216     count= max<ulong>(count, h->m_min_number_of_tables);
217     p->m_table_share_sizing= apply_load_factor(count, h->m_load_factor_static);
218   }
...
336 }
{{< /highlight >}}

This indicates that the share value, p->m_hints.m_table_definition_cache is used to determine the size passed to pfs_malloc.
Looking more closely, the ulongs defined at the start of the apply_heuristic function correspond to the MySQL system variables for max_connections, table_open_cache, table_definition_cache, and open_files_limit.

Now we know that we can lower the memory pressure of the largest allocation by decreasing the size of table_definition_cache.
We can perform a similar exercise for the other large allocations to determine which of these four sysvars affect them.

# Step 4: Apply a Suitable Fix

With the relevant sysvars available, we can try tuning our server with performance_schema enabled.

### Old
{{< highlight bash >}}
$ mysql -Bse "SELECT @@max_connections, @@table_open_cache, @@table_definition_cache, @@open_files_limit"
151     2000    1400    5000
$ ps -o rss $(pidof mysqld)
  RSS
456472
{{< /highlight >}}

### New
{{< highlight bash >}}
$ mysql -Bse "SELECT @@max_connections, @@table_open_cache, @@table_definition_cache, @@open_files_limit"
50      500     500     2000
$ ps -o rss $(pidof mysqld)
  RSS
102608
{{< /highlight >}}

Assuming that our workload can support these lowered system variables, we now have an acceptable solution.
If it cannot, then we need to look at changing the memory allocated to this device.

# Conclusion

Standard sysadmin tools such as *free*, *ps*, and *pmap* are excellent for triaging memory issues.
To get a better understanding of the memory a particular application is using, Valgrind's *massif* tool is useful to provide a statistical breakdown of heap allocations.

While these tools explain **what** happens, familiarity with *gdb* and the application source code is often necessary to understand **why** something happens.
Once we know why something happens we are now equipped to solve the problem either by mitigating the behavior (as we did in this case) or submitting a bug report / patch for the behavior if we believe it is incorrect.

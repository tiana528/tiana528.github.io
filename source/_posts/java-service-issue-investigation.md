---
title: java service issue investigation
date: 2020-02-08 12:33:36
tags:
---

# cpu
## top
```
ps aux |grep java
top -H -p pid # -H : check threads cpu usage
top # check processes cpu usage
```

# memory
## free
```
free -m
              total        used        free      shared  buff/cache   available
Mem:           7982         830         437           3        6714        6795
Swap:          1023         131         892
```
## dmesg
If process is killed by the OS due to OOM, we can get logs from here.
```
sudo dmesg | grep -i kill 
```

## vmstat
Check memroy, cpu, io ...
```
vmstat 2 10 -t # 2 : interval, 10 : times

procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu----- -----timestamp-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st                 UTC
 3  0 134548 400884 152320 6737836    0    0     3    54    3    8  5  1 94  0  0 2020-02-08 08:35:06
 0  0 134548 401940 152320 6737808    0    0     0    54 1309 1883  1  1 98  0  0 2020-02-08 08:35:08


# Procs
#   r: The number of processes waiting for run time.
#   b: The number of processes in uninterruptible sleep.
# Memory
#   swpd: the amount of virtual memory used. (is the value is larger than 0, it means the memory is not enough)
#   free: the amount of idle memory.
#   buff: the amount of memory used as buffers.
#   cache: the amount of memory used as cache.
#   inact: the amount of inactive memory. (-a option)
#   active: the amount of active memory. (-a option)
# Swap
#   si: Amount of memory swapped in from disk (/s).
#   so: Amount of memory swapped to disk (/s).
#   (If si and so is not 0 for a while, it means memory is not enough)
# IO
#   bi: Blocks received from a block device (blocks/s).
#   bo: Blocks sent to a block device (blocks/s).
#   (if bi+bo>1000, and wa value is large, then IO is bottleneck)
# System
#   in: The number of interrupts per second, including the clock.
#   cs: The number of context switches per second.
# CPU
#   These are percentages of total CPU time.
#   us: Time spent running non-kernel code. (user time, including nice time)
#   sy: Time spent running kernel code. (system time)
#   id: Time spent idle. Prior to Linux 2.5.41, this includes IO-wait time.
#   wa: Time spent waiting for IO. Prior to Linux 2.5.41, included in idle.
#   st: Time stolen from a virtual machine. Prior to Linux 2.6.11, unknown.

```

# disk
## df
```
df -h

Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           799M  4.0M  795M   1% /run
/dev/xvda1       31G   16G   16G  49% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/md0         80G  2.5G   78G   4% /mnt
tmpfs           799M     0  799M   0% /run/user/21363
```

```
du -m /mnt | sort -rn | head -3

2413	/mnt
1389	/mnt/abc
1322	/mnt/abc/def
```


# network
## netstat
```
netstat -nat | awk '{print $6}' | sort | uniq -c | sort -rn # check socket status
```



# java process
## jstack
Check status status of a java process
```
sudo jstack -F 1423 # 1423 is process id
```

## jinfo
```
sudo jinfo -flags 13474

Attaching to process ID 13474, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.152-b16
Non-default VM flags: -XX:CICompilerCount=2 -XX:+CMSParallelRemarkEnabled -XX:InitialHeapSize=132120576 -XX:+ManagementServer -XX:MaxHeapSize=1073741824 -XX:MaxNewSize=174456832 -XX:MaxTenuringThreshold=6 -XX:MinHeapDeltaBytes=196608 -XX:NewSize=44040192 -XX:OldPLABSize=16 -XX:OldSize=88080384 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseParNewGC
Command line:  -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18288 -Xmx1024m -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled

```
## jmap
check heap usage status of a java process
```
sudo jmap -heap 13474
Attaching to process ID 13474, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.152-b16

using parallel threads in the new generation.
using thread-local object allocation.
Concurrent Mark-Sweep GC

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 1073741824 (1024.0MB)
   NewSize                  = 44040192 (42.0MB)
   MaxNewSize               = 174456832 (166.375MB)
   OldSize                  = 88080384 (84.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 39649280 (37.8125MB)
   used     = 5273360 (5.0290679931640625MB)
   free     = 34375920 (32.78343200683594MB)
   13.300014527376034% used
Eden Space:
   capacity = 35258368 (33.625MB)
   used     = 4555320 (4.344291687011719MB)
   free     = 30703048 (29.28070831298828MB)
   12.919826578473513% used
From Space:
   capacity = 4390912 (4.1875MB)
   used     = 718040 (0.6847763061523438MB)
   free     = 3672872 (3.5027236938476562MB)
   16.352867012593283% used
To Space:
   capacity = 4390912 (4.1875MB)
   used     = 0 (0.0MB)
   free     = 4390912 (4.1875MB)
   0.0% used
concurrent mark-sweep generation:
   capacity = 88080384 (84.0MB)
   used     = 45554184 (43.44385528564453MB)
   free     = 42526200 (40.55614471435547MB)
   51.71887534005301% used
```

## jstat
```
sudo jstat  -gc 13474 2000 10 
# 13474 : process id, 2000 : print for every 2000 second, 10 : print 10 times

 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
4288.0 4288.0  0.0   755.6  34432.0  11079.1   86016.0    44522.9   57912.0 56542.0 7064.0 6771.9    221    1.105   4      0.035    1.140
4288.0 4288.0  0.0   755.6  34432.0  11124.5   86016.0    44522.9   57912.0 56542.0 7064.0 6771.9    221    1.105   4      0.035    1.140
4288.0 4288.0  0.0   755.6  34432.0  11152.6   86016.0    44522.9   57912.0 56542.0 7064.0 6771.9    221    1.105   4      0.035    1.140
4288.0 4288.0  0.0   755.6  34432.0  11737.6   86016.0    44522.9   57912.0 56542.0 7064.0 6771.9    221    1.105   4      0.035    1.140

# S0C	Current survivor space 0 capacity (KB).
# S1C	Current survivor space 1 capacity (KB).
# S0U	Survivor space 0 utilization (KB).
# S1U	Survivor space 1 utilization (KB).
# EC	Current eden space capacity (KB).
# EU	Eden space utilization (KB).
# OC	Current old space capacity (KB).
# OU	Old space utilization (KB).
# PC	Current permanent space capacity (KB).
# PU	Permanent space utilization (KB).
# YGC	Number of young generation GC Events.
# YGCT	Young generation garbage collection time.
# FGC	Number of full GC events.
# FGCT	Full garbage collection time.
# GCT	Total garbage collection time.
```

# jps
Check all java process information
```
sudo jps -mlvV 
# m : print main method argument
# l : print whole package name, main class, jar full path
# v : print jvm parameter
# V : print JVM parameter passed by flag file
```





# others
## tail
```
tail -100f server.log #  Show last 100 lines and show new lines in realtime
```
## awk
```
awk '{print $1,$2}' a.txt
awk '{print NR,$0}' a.txt # NR is number of records, usually equal to line number
```

## find
```
find /tmp/ /user/ -name *.log
find . -iname *.txt # iname : case insensitive
find . -type d # get all sub-dir of current dir
find . -type l # get all symlkink

```


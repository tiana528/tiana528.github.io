---
title: resilient architecture should also consider hard shutdown
date: 2019-04-14 19:39:06
tags:
---

# Graceful shutdown and hard shutdown
When a program/service is shutdown unexpectedly, it may be graceful shutdown or hard shutdown.
When a service is killed with signal 15, if the service catches the signal and perform some cleanup before exit, it is graceful shutdown.
When a service is killed with 9, the service cannot catch the signal and will be termidated immediately, this is hard shutdown.

# hard shutdown is unavoided
You cannot prevent your service from hard shutdown, it may happen due to out of memory, disk full, operating system kernal crash, or even instance power off.
 
Many systems have graceful shutdown hooks to do some clean up logic during unexpected shutting down, but it is not enough.

# how to create resilient architecture
Your service should be strong enough(still keep consistency, be able to recover) no matter the service is killed by 9 or by 15.
It means that you need to make the service handle graceful shutdown, which is quite straightforward, just adding a shutdown hook to do some cleanups.
It is not easy to make the service support the case of hard shutdown, many application side logic is needed here and there to support it.

# some examples of high resilient architecture which supports hard shutdown
One good example is mysql, it writes operation logs, and if the mysql process hard shutdown happens, it can still recover from the failure, and won't have any inconsistency.
Another example is hadoop, a hadoop cluster may have thousands of nodes, and it can still work properly even if any one node is lost, the data is already duplicated on other nodes, the failed tasks will be retried on other nodes.



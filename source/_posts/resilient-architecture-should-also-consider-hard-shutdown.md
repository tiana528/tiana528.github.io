---
title: resilient architecture should also consider hard shutdown
date: 2019-04-14 19:39:06
tags:
---

# graceful shutdown and hard shutdown
When a program/service is shutdown unexpectedly, it may be graceful shutdown or hard shutdown.
When a service is killed with signal 15, if the service catches the signal and perform some cleanup before exit, it is graceful shutdown. If the service does not catch the signal and exits intermediately, it is hard shutdown.
When a service is killed with 9, the service cannot catch the signal and will be termidated immediately, this is also hard shutdown.

# hard shutdown is unavoided
You cannot prevent your service from hard shutdown, it may happen due to out of memory, disk full, operating system kernal crash, or even instance power off.
 
# how to create resilient architecture
Your service should be strong enough(still keep consistency, be able to recover) no matter the service is killed by 9 or by 15.
It means that you need to make the service handle graceful shutdown, which is quite straightforward, just adding a shutdown hook to do some cleanups.
It is not easy to make the service support the case of hard shutdown, many application side logic is needed here and there to support it.
**But it is very important to handle the case of hard shutdown to achieve high resilient architecture. If your data/service will be inconsistent and there is no way to recover whenever hard shutdown happens, it might just be unacceptable.**

# some examples of high resilient architecture which support hard shutdown
One good example is mysql, it writes operation logs, and if the mysql process hard shutdown happens, it can still recover from the failure, and won't have any inconsistency.
Another example is hadoop, a hadoop cluster may have thousands of nodes, and it can still work properly even if any one node is lost, the data is already duplicated on other nodes, the failed tasks will be retried on other nodes.



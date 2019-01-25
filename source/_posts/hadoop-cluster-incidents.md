---
title: hadoop cluster incidents
date: 2019-01-25 20:03:09
tags: hadoop
---

# Purpose

The operation of Hadoop cluster is not easy. There are many parameters and there is no perfect configuration that fit all situations. Monitoring and learn from each incidents is very important. This article aims at summarizing some incidents and they can be seen as valued knowledge to help investigate future incidents and discuss approaches to prevent. 


# datanode Xmx configured does not fit the instance

- situation
    - The whole cluster performance degrades.
    - One node in the cluster has extremely higher CPU utilization than other nodes.
    - The datanode process on that node takes up more than 300% CPU by checking top command.
    - The accumulate GC time of the datanode processs on that node increases rapidly(know it through datadog jvm.gc.parnew.time metrics).
- analysis
    - The cluster performance degrades because one datanode process behaves bad.
    - GC time increases rapidly indicates the memory is insufficient.
- root cause
    - The Xmx parameter of the datanode process is configured too large. The slave instance type is aws c4.2xlarge which is 15GB, 8vCPU. Each core(map,reduce,application master) Xmx is configured as 2GB, node manager Xmx is also 2GB, but datanode JMX is configured as 10GB. Meaning that the datanode Xmx configured is much higher than the free memory on the node thus causing frequent memory swap and performance degradation, as well as longer GC time. This articles explains a little more about [the situation when a large heap(Xmx) is configured](http://www.javaperformancetuning.com/news/qotm045.shtml).
- learn
    - Monitoring GC time increasing speed helps find the situations that the JVM paramers are mis-configured. Free space on the node should be considered when deciding the Xmx value.

# to be continued


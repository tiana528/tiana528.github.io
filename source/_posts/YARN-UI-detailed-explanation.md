---
title: YARN UI detailed explanation
date: 2019-02-11 12:14:44
tags: hadoop
---

# Description
YARN UI url is {hadoop_master_ip:8088}. There is a lot of information in this UI as follows which is very helpful for analyzing issues. This article aims at explaning everything in detail.
```
Cluster
    About
    Nodes
    Node Labels
    Applications
        NEW
        NEW SAVING
        SUBMITTED
        ACCEPTED
        RUNNING
        FINISHED
        FAILED
        KILLED
    SCHEDULER
TOOLS
    CONFIGURATION
    LOCAL LOGS
    SERVER STACKS
    SERVER METRICS
```

hadoop version is 2.7.3.

# Cluster
## Common Header

- Cluster Metrics
```
Apps Submitted : 3220
Apps Pending : 1
Apps Running : 3
Apps Completed : 3216
Containers Running : 3
Memory Used : 3 GB
Memory Total : 9 GB
Memory Reserved : 0 B
VCores Used : 3
VCores Total : 9
VCores Reserved : 0
Active Nodes : 3
Decommissioned Nodes : 0
Lost Nodes : 0
Unhealthy Nodes	: 0
Rebooted Nodes : 0
```
    - source code
        - [MetricsOverviewTable](https://github.com/apache/hadoop/blob/a55d6bba71c81c1c4e9d8cd11f55c78f10a548b0/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager/src/main/java/org/apache/hadoop/yarn/server/resourcemanager/webapp/MetricsOverviewTable.java#L68-L78)
        - [ClusterMetricsInfo](https://github.com/apache/hadoop/blob/a55d6bba71c81c1c4e9d8cd11f55c78f10a548b0/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager/src/main/java/org/apache/hadoop/yarn/server/resourcemanager/webapp/dao/ClusterMetricsInfo.java)
    - Apps Submited:
        - Total apps [submitted](https://github.com/apache/hadoop/blob/84e22a6af46db2859d7d2caf192861cae9b6a1a8/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager/src/main/java/org/apache/hadoop/yarn/server/resourcemanager/scheduler/QueueMetrics.java#L60). When an application is submitted, it increases. This count never decreases.
    - Apps pending
        - Total apps in the pending status. When an application attempt is submitted, this count inreases. When an application attempt becomes running/finished/... from the pending status, this count decreases.
        - An application(RMApp) can have multiple app attempts based on [yarn.resourcemanager.am.max-attempts](https://hadoop.apache.org/docs/r2.4.1/hadoop-yarn/hadoop-yarn-common/yarn-default.xml).
    - Apps running
        - Total apps in the running status. When an application attempt becomes running, this count increases. When an application attempt becomes finished/... from the running status, this count decreases.
    - Apps Completed
        - Total apps completed. Completed(successfully) + failed + killed.
    - Containers Running
        - Total running container number. When containers are allocated, this count increases, when containers are released, this count decreases.
    - Memory Used
        - Currently allocated memory. When containers are allocated and released, this count changes accordingly, basicly change value is container_size * container_number.
    - Memory Total
        - Total memory of the cluster. It is available memory + allocated memory.
    - Memory reserved
        - Currently reserved memory. This [article](https://blog.cloudera.com/blog/2013/06/improvements-in-the-hadoop-yarn-fair-scheduler/) explains the usage of reserved memory very well. It can be used to prevent an application from being starvation.
    - Vcores Used/Total/Reserved
        - This is similiar as memory.
    - Active Nodes
        - Number of nodes in the active status.
    - Decommissioned Nodes
        - Number of nodes in the decommissioning status. For more detailed info about decommission a node, refer [this](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.5/bk_administration/content/ref-a179736c-eb7c-4dda-b3b4-6f3a778bd8c8.1.html).
    - Lost nodes
        - Number of nodes lost. Node has not sent a heartbeat for some configured time threshold are considered as lost.
    - Unhealthy nodes
        - Number of nodes that are unhealthy. Node manager sends heartbeat to the resource manager regularly and the heartbeat contains the healthy report of the node, then the resource manager can know which nodes are unhealthy.
    - Rebooted nodes
        - Number of nodes that are rebooted.

## About
```
Cluster ID:	1549960821363
ResourceManager state:	STARTED
ResourceManager HA state:	active
ResourceManager HA zookeeper connection state:	ResourceManager HA is not enabled.
ResourceManager RMStateStore:	org.apache.hadoop.yarn.server.resourcemanager.recovery.LeveldbRMStateStore
ResourceManager started on:	Tue Feb 12 08:40:21 +0000 2019
ResourceManager version:	2.7.3-...
Hadoop version:	2.7.3-...
```

## Nodes

## Applications

## Scheduler

# Tools

## configuration

## Local logs

## Server

## stacks

## Server

## metrivs

# Check job detailed information

## running jobs

## finished jobs



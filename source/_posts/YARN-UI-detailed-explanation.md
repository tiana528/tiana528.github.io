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
- Scheduler Metrics
```
Scheduler Type : Fair Scheduler
Scheduling Resource Type : [MEMORY, CPU]
Minimum Allocation : <memory:256, vCores:1>	
Maximum Allocation : <memory:61440, vCores:32>
```
    - Scheduler Type
        - The scheduler used by the resource manager.
    - Scheduling Resource Type
        - Get all resource types information from known resource types.
    - Minimum Allocation
        - Get minimum allocatable resource.
    - Maximum Allocation
        - Get maximum allocatable resource at the cluster level.

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
- Cluster ID
    - This is the start timestamp of the resource manager.
- ResourceManager state
    - The state of the resource manager, which might be one of NOTINITED, INITED, STARTED, STOPPED.
- ResourceManager HA state
    - HA state of resource manager
- ResourceManager HA zookeeper connection state
- ResourceManager RMStateStore
    - RMStateStore is storage of ResourceManager state. Takes care of asynchronous notifications and interfacing with YARN objects. Real store implementations need to derive from it and implement blocking store and load methods to actually store and load the state.
- ResourceManager started on
    - The start timestamp of resource manager
- ResourceManager version
    - The version of resource manager
- Hadoop version
    - The version of hadoop

## Applications
This shows a table of applications. One example row is as follows.
```
ID : application_1549960821363_10810
User : 1
Name : job1...
Application Type: MAPREDUCE
Queue : root.q1
Starttime : Thu Feb 14 22:00:21 +0900 2019
Finishtime : Thu Feb 14 22:00:34 +0900 2019
State : FINISHED
FinalStatus : SUCCEEDED
Running Containers : N/A
Tracking UI : History
Blacklisted Nodes : N/A
```
- source code
    - [AppsBlock](https://github.com/apache/hadoop/blob/88625f5cd90766136a9ebd76a8d84b45a37e6c99/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-common/src/main/java/org/apache/hadoop/yarn/server/webapp/AppsBlock.java#L150)
- ID
    - ApplicationId represents the globally unique identifier for an application. The globally unique nature of the identifier is achieved by using the cluster timestamp i.e. start-time of the ResourceManager along with a monotonically increasing counter
 for the application.
- User
    - The user who submitted the application.
- Name
    - Get the user-defined name of the application.
- State
    - Enumeration of various [states of an ApplicationMaster](https://github.com/apache/hadoop/blob/a55d6bba71c81c1c4e9d8cd11f55c78f10a548b0/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-api/src/main/java/org/apache/hadoop/yarn/api/records/YarnApplicationState.java).
- Running Containers
    - The number of containers.
- Tracking UI
    - The tracking url for the application.
- Blacklisted Nodes
    - A blacklisted node to indicate that a node is unhealthy and hadoop should not assign any tasks to it anymore.

## Scheduler
This is a graph showing the current usage status of queues. 

Take fair scheduler as a example. This graph tells us what are the queues, how much resource can a queue use up on fair, how much resource of a queue has been used up, is a queue used more than fair/less than fair, which queue is busy/idle, and so on.


# Tools

## Configuration
This is actually redirects to http://{hadoop-master-ip}:8088/conf

This shows all configurations from the source of hdfs-default.xml, yarn-default.xml, mapred-default.xml, core-default.xml, hdfs-site.xml, yarn-site.xml, mapred-site.xml, core-site.xml,


## Local logs
yarn logs

## Server stacks
Stack trace of the resource manager.

## Metrics
JMX metrics of the resource manager.

# Check job detailed information
This is a complex part, it is not as straightforward as other parts because there are many logs and UIs involved.
Yarn log aggregation needs to be enabled for checking the logs. ([Enable yarn log aggregation](https://tiana528.github.io/2018/11/19/hive-configuration-best-practise/))


## Check application detailed info
Click application id such as `application_1542608710651_1268349`, then the application UI will open. URL is : {master_ip:8088}/cluster/app/application_1549418419376_658006

The bottom part shows the application attempts if there are any. If an application attemp fails, another application attempt will be triggered until the attempt number meets the configured maximum value. We can know how many times of the application attempts have been tried/trying from this UI.

It is able to check the logs of the corresponding application master by clicking the `logs` in each row.


## Check application attempt detailed info
Click `Tracking URL: ApplicationMaster`
- If the application master hasn't been started, the UI will be redirected to the application detailed info UI.
- If the application master is running, the URL is like : {master_ip}:8088/proxy/application_1549418419376_657056/mapreduce/job/job_1549418419376_657056
```
Task Type :  Map, Reduce
Progress: ...
Total  : 2434, 999
Pending : 0, 871
Running : 0, 63
Complete : 2434, 65

Attempt Type : Maps, Reduces
New : 0, 871
Running : 0, 63
Failed : 0, 0
Killed : 0, 0
Successful : 2434, 65
```
    - The above part(Task type) shows the total number of map/reduce tasks and how many are running/pending/completed.
    - The blow part(Attempt type) shows the info of attempts. How many attemps are new/running/failed/killed/successful. 
- If the application master has finished, the UI will be redirected to jobhistory. The URL is like : {job_history_server_ip}:19888/jobhistory/job/job_1549418419376_658293. 
```
Task Type :  Map, Reduce
Total  : 12328, 0
Complete : 0, 0

Attempt Type : Maps, Reduces
Failed : 597, 0
Killed : 209, 0
Successful : 0, 0
```
    - The above part(Task type) shows the total number of map/reduce tasks and how many have been finished.
    - The blow part(Attempt type) shows the info of attempts. How many attemps are failed/killed/successful. 

## Check task attempt detailed information
Click the links in the jobs UI, then it will redirect to a list of tasks, and click task_id(e.g. task_1549418419376_657056_r_000732), the detailed info UI of a task attempt will open.
```
Attempt : attempt_1549418419376_657056_r_000732_0
Progress : 76.90
State: Running
Status : reduce > reduce
Node : 172.1.1.2:8042
Logs : logs
Started : Sat Feb 16 11:57:37 +0900 2019	
Finished : N/A
Elapsed : 42sec
Note : 
Actions : kill
```
   - Attempt : attempt id
   - Progress : the progress of the current task attempt
   - State : the state of the current task attempt
   - Status : what is the status of the current task attempt
       - If it is a map task attempt, it will show `SCHEDULED`, `...> map`, `... > sort`. They are different phrases of the map process.
       - If it is a reducer task attempt, it will show `SCHEDULED`, `35 / 35 copied.`, `reduce > sort	`, `reduce > reduce`. They are different phrases of the reduce process.
   - Logs : 
```
stderr : Total file length is 243 bytes.
stdout : Total file length is 0 bytes.
syslog : Total file length is 79129 bytes.
```




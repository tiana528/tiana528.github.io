---
title: How to fast fail hive jobs
date: 2018-11-15 23:37:45
tags:
    - hadoop
    - hive
categories:
    - big data

---


## background
When running hive jobs in hadoop clusters on mapreduce, we always set the limitation of how much local and hdfs disk a job can use at most. Such limitation is per job basis and it can prevent a single job from using up too much disk resource and causing a node or a cluster unstable.

During running one job, if any task belong to the job finds the limitation is exceeded, the task will fail and at such time, we want to **fast fail the job instead of retrying the failed tasks many times**. Because since the local or hdfs max disk usage limitation is reached, the retry will probably still fail and is not much helpful.

## appoaches
I will introduce how to configure to fast fail hive jobs when the local or hdfs limitation is exceeded.
### fast fail a job when too much *local disk* is used in one node

- **hadoop configuration**
```xml
<property>
  <name>mapreduce.job.local-fs.single-disk-limit.bytes</name>
  <value>322122547200</value>
</property>
<property>
  <name>mapreduce.job.local-fs.single-disk-limit.check.kill-limit-exceed</name>
  <value>true</value>
</property>
```

- **implementation detail**
A task(org.apache.hadoop.mapred.Task) contains a thread checking all local working directories(mapreduce.cluster.local.dir), and will fast fail the job if any local working directory exceeds the limitation.

- **required hadoop version**
3.1.0 or apply patch [MAPREDUCE-7022](https://issues.apache.org/jira/browse/MAPREDUCE-7022)



### fast fail a job when too much *hdfs disk* is used in the cluster

- **configuration**
 - **hive configuration**
Set hive config hive.exec.scratchdir to per job basis, and set the quota limitation for the path.
 - **hadoop configuration**
```xml
<property>
  <name>mapreduce.job.dfs.storage.capacity.kill-limit-exceed</name>
  <value>true</value>
</property>
```
- **internal implementation**
A subclass of ClusterStorageCapacityExceededException will be thrown if the cluster capacity limitation is exceeded and then YarnChild will fast fail the job.

- **required hadoop version**
3.3.0 or apply patch [MAPREDUCE-7148](https://issues.apache.org/jira/browse/MAPREDUCE-7148)





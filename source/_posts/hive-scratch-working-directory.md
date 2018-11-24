---
title: hive scratch directory
date: 2018-11-17 17:57:51
tags: 
    - hive
    - hadoop
categories:
    - big data
---
This article aims at explaining hive scratch directory.

## Scratch directory usage
Hive scratch directory is a temporary working space for storing the plans for different map/reduce stages of the query as well as the intermediate outputs of these stages.
## Scratch directory clean up
Hive scratch directory is usually cleaned up by the hive client when the query finishes. However, some data may be left behind if hive client terminates abnormally. Hive server2 contains a thread ([ClearDanglingScratchDir](https://github.com/apache/hive/blob/f37c5de6c32b9395d1b34fa3c02ed06d1bfbf6eb/ql/src/java/org/apache/hadoop/hive/ql/session/ClearDanglingScratchDir.java#L43-L55)) to clean up the remaining files, we can also write our own script to do the clean up if not running Hive server2.

## Scratch directory types
Hive queries may be procesed in local(the instance which hive client is invoked) or in remote(hadoop cluster). There also have two kinds of scratch dir accordingly, one in local, the other in hdfs.

## Scratch directory configuration
*hive.exec.local.scratchdir* for local and *hive.exec.scratchdir* for HDFS([hive configuration](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties)).

Note that since hive 0.14.0, the HDFS scratch directory created will be *${hive.exec.scratchdir}\\${user_name}* indicating it supports multi-tenant natively and there is no need to include user_id in the value.

## Scratch directory example
We run a simple query and see what are the files generated in the scratch directory.
```sql
--@INTERNAL hive_version:hive2
select count(*) from tb1 
where col1>0
group by col2
```
- when query is submitted to the cluster and waiting for containers
```
drwxr-xr-x ${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-1/-mr-10000/.hive-staging_hive_2018-11-18_11-32-47_530_1102432233025308705-1/_tmp.-ext-10001
-rw-r--r-- ${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-2/-mr-10004/8cf23c5d-9a81-4e99-ae69-d8b99eee1a08/map.xml
-rw-r--r-- ${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-2/-mr-10004/8cf23c5d-9a81-4e99-ae69-d8b99eee1a08/reduce.xml
```
    - -ext- : a dir indicates the final query output
    - -mr- : a output directory for each MapReduce job
    - map.xml : map plan
    - reduce.xml : reduce plan

- when query is running in the hadoop cluster
```
-rw-r--r-- ${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-1/-mr-10000/.hive-staging_hive_2018-11-18_11-32-47_530_1102432233025308705-1/-ext-10001/000000_0
```
    - data is generated in the -ext- dir  
- when query finished, scratch dir with all files are cleaned up

## Scratch directory related INFO logs
These info logs are generated when running the query in the previous section.

```
session.SessionState: Created HDFS directory: /${hive.exec.scratchdir}/${job_id}/${user_name}
session.SessionState: Created HDFS directory: /${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4
session.SessionState: Created HDFS directory: /${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/_tmp_space.db
ql.Context: New scratch dir is hdfs://${namenode_ip}:8020$/{hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-1
common.FileUtils: Creating directory if it doesn't exist: hdfs://${namenode_ip}:8020/${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-1/-mr-10000/.hive-staging_hive_2018-11-18_11-32-47_530_1102432233025308705-1
ql.Context: New scratch dir is hdfs://${namenode_ip}:8020/${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-2
exec.Utilities: PLAN PATH = hdfs://${namenode_ip}:8020/${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-2/-mr-10004/8cf23c5d-9a81-4e99-ae69-d8b99eee1a08/map.xml
exec.Utilities: PLAN PATH = hdfs://${namenode_ip}:8020/${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-2/-mr-10004/8cf23c5d-9a81-4e99-ae69-d8b99eee1a08/reduce.xml
exec.FileSinkOperator: Moving tmp dir: hdfs://${namenode_ip}:8020/${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-1/-mr-10000/.hive-staging_hive_2018-11-18_11-32-47_530_1102432233025308705-1/_tmp.-ext-10001 to: hdfs://${namenode_ip}:8020/${hive.exec.scratchdir}/${job_id}/${user_name}/bf17195d-b591-457a-a5e1-28426156c7f4/hive_2018-11-18_11-32-47_530_1102432233025308705-1/-mr-10000/.hive-staging_hive_2018-11-18_11-32-47_530_1102432233025308705-1/-ext-10001
```
- First several HDFS scratch directories are created during start [SessionState]((https://github.com/apache/hive/blob/840dd431f3772772fc57060e27e3f2bee72a8936/ql/src/java/org/apache/hadoop/hive/ql/session/SessionState.java#L698-L704).

- _hive.hdfs.session.path = ${hive.exec.scratchdir}/${job_id}/${user_name}/${hive.session.id}
- hive.exec.plan = ${hive.exec.scratchdir}/${job_id}/${user_name}/${hive.session.id}/${context execution id}-${task runner id}/-mr-${path id}/${random uuid}
    - [hive.session.id](https://github.com/apache/hive/blob/840dd431f3772772fc57060e27e3f2bee72a8936/ql/src/java/org/apache/hadoop/hive/ql/session/SessionState.java#L414)
    - [context execution id](https://github.com/apache/hive/blob/b4302bb7ad967f15ca1b708685b2ac669e3cf037/ql/src/java/org/apache/hadoop/hive/ql/Context.java#L313)
    - [task running id](https://github.com/apache/hive/blob/b4302bb7ad967f15ca1b708685b2ac669e3cf037/ql/src/java/org/apache/hadoop/hive/ql/Context.java#L491)
    - [path id](https://github.com/apache/hive/blob/b4302bb7ad967f15ca1b708685b2ac669e3cf037/ql/src/java/org/apache/hadoop/hive/ql/Context.java#L685)
    - [random uuid](https://github.com/apache/hive/blob/6d713b6564ecb9d1ae0db66c3742d2a8bc347211/ql/src/java/org/apache/hadoop/hive/ql/exec/Utilities.java#L668)
- map.xml path = ${hive.exec.plan}/map.xml
- reduce.xml path = ${hive.exec.plan}/reduce.xml




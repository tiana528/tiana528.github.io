---
title: hadoop ui detailed description
date: 2019-02-09 23:21:33
tags:
---

# HDFS UI
Open HDFS UI by browser, the URL is {hadoop_master_ip:50070}.
Ther are a lot of information, and we will explain one by one.
This article is based on hadoop of version 2.7.3.

- Overview
    - Overview
```
Started:	Sun Feb 10 09:33:34 UTC 2019
Version:	2.7.3, ...
Compiled:	2018-11-16T09:23Z by root from ...
Cluster ID:	CID-c489b321-4d13-423b-a7e9-e8f66355e17a
Block Pool ID:	BP-5769292-172.18.204.140-1549791212026
```
        - Source code
            - [dfshealth.html](https://github.com/apache/hadoop/blob/release-2.7.3-RC2/hadoop-hdfs-project/hadoop-hdfs/src/main/webapps/hdfs/dfshealth.html)
        - Explanation
            - Started
                - The starttime of the namenode.
            - Version
                - Hadoop version.
            - Compiled
                - Haodop compiled information.
            - Cluster ID
                - As explained in (hdfs federation)[https://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-hdfs/Federation.html], a ClusterID identifier is used to identify all the nodes in the cluster. When a Namenode is formatted, this identifier is either provided or auto generated.
                - When formatting the namenode of an running hadoop cluster, a new ClusterID will be generated, it is also needed to change the `usr/local/hadoop/dfs/datanode/current/VERSION` file to update the value of ClusterID on all slaves. 
            - Block Pool ID:
                - As explained in (hdfs federation)[https://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-hdfs/Federation.html], a Block Pool is a set of blocks that belong to a single namespace.
    - Summary
```
Security is off.

Safemode is off.

181 files and directories, 60 blocks = 241 total filesystem object(s).

Heap Memory used 254 MB of 401.13 MB Heap Memory. Max Heap Memory is 9.94 GB.

Non Heap Memory used 59.1 MB of 60.06 MB Commited Non Heap Memory. Max Non Heap Memory is -1 B.

Configured Capacity:	899.56 GB
DFS Used:	2.02 GB (0.23%)
Non DFS Used:	3.97 GB
DFS Remaining:	893.56 GB (99.33%)
Block Pool Used:	2.02 GB (0.23%)
DataNodes usages% (Min/Median/Max/stdDev):	0.06% / 0.31% / 0.31% / 0.12%
Live Nodes	3 (Decommissioned: 0)
Dead Nodes	0 (Decommissioned: 0)
Decommissioning Nodes	0
Total Datanode Volume Failures	0 (0 B)
Number of Under-Replicated Blocks	0
Number of Blocks Pending Deletion	2
Block Deletion Start Time	2/10/2019, 6:33:34 PM
```
        - Security
            - Whether hadoop [security mode](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SecureMode.html) is on or off. When Hadoop is configured to run in secure mode, each Hadoop service and each user must be authenticated by Kerberos.
        - Safemode
            - Whether [safemode](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#Safemode) is on or off. Safemode for the NameNode is essentially a read-only mode for the HDFS cluste
            - 
        - 181 files
            - The current size of inodes in HDFS. [Link](https://github.com/apache/hadoop/blob/45caeee6cfcf1ae3355cd880402159cf31e94a8a/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/namenode/FSNamesystem.java#L4844)
        - 60 blocks
            - The current number of blocks. [Link](https://github.com/apache/hadoop/blob/45caeee6cfcf1ae3355cd880402159cf31e94a8a/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/namenode/FSNamesystem.java#L6017)
        - Memory related
            - This part describes namenode in the following format.
                - used {used} MB of {committed} MB. Max Memory is {max}
            - Refer to [MemoryUsage](https://docs.oracle.com/javase/8/docs/api/java/lang/management/MemoryUsage.html) for the definition of used, committed, and max.

# Yarn UI





1、HDFS页面：50070

2、YARN的管理界面：8088

3、HistoryServer的管理界面：19888

http://hadooptutorial.info/hdfs-web-ui/


https://www.maiyewang.com/?p=10785
https://datameer.zendesk.com/hc/en-us/articles/115005289466-How-to-Enable-Tez-History-UI-for-Hadoop-

http://d.hatena.ne.jp/ksmemo/20110613/p1

https://www.oreilly.com/library/view/hadoop-operations-and/9781782165163/ch04s09.html

https://www.dezyre.com/questions/5227/how-to-find-files-in-hadoop-web-ui

https://www.quora.com/Whats-a-good-GUI-for-Hadoop-What-are-the-options-for-the-open-source-end-user-interfaces-for-Hadoop-other-than-HUE

https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-ui/dependency-analysis.html


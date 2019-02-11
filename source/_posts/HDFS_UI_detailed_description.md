---
title: HDFS UI detailed description
date: 2019-02-09 23:21:33
tags: hadoop
---

# HDFS UI
Open HDFS UI by browser, the URL is {hadoop_master_ip:50070}.

This article aims at explaining everything in the HDFS Overview UI in detail, other tabs content is self-described.

This article is based on hadoop of version 2.7.3.

(A lot of information can be get from FSNameSystem.)

## Overview
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
        - As explained in [hdfs federation](https://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-hdfs/Federation.html), a ClusterID identifier is used to identify all the nodes in the cluster. When a Namenode is formatted, this identifier is either provided or auto generated.
        - When formatting the namenode of an running hadoop cluster, a new ClusterID will be generated, it is also needed to change the `usr/local/hadoop/dfs/datanode/current/VERSION` file to update the value of ClusterID on all slaves. 
    - Block Pool ID:
        - As explained in [hdfs federation](https://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-hdfs/Federation.html), a Block Pool is a set of blocks that belong to a single namespace.

## Summary

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
    - This part describes namenode memory information in the following format.
        - used {used} MB of {committed} MB. Max Memory is {max}
    - Refer to [MemoryUsage](https://docs.oracle.com/javase/8/docs/api/java/lang/management/MemoryUsage.html) for the definition of used, committed, and max.
    - init is near xms, max is near xmx, used is actual being used, committed is garranteed by os. committed >= used. There can be OutOfMemoryError if max > used + allocating_memory > committed, when os virtual memory is insufficient.
- Configured Capacity
    - Total raw capacity of data nodes in bytes. [Link](https://github.com/apache/hadoop/blob/a55d6bba71c81c1c4e9d8cd11f55c78f10a548b0/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/blockmanagement/DatanodeStats.java#L36)
- DFS Used
    - The percentage of the used capacity over the total capacity.
- Non DFS Used
    - The total used space by data nodes for non-DFS purposes such as storing temporary files on the local file system.
- DFS Remaining
    - The remaining capacity.
- Block Pool Used
    - The used space by the block pool on all data nodes.
- DataNodes usages
    - Min/Median/Max/stdDev refers to the minimum/median/maximum/standard_deviation of the dfs used space percentage by all datanodes.
- Live Nodes
    - Number of datanodes which are currently live.
- Dead Nodes
    - Number of datanodes which are currently dead.
- Decommissioning Nodes
    - Number of datanodes where decommissioning is in progress.
- Total Datanode Volume Failures
    - Total number of volume failures across all Datanodes.
- Number of Under-Replicated Blocks
    - Get aggregated count of all blocks with low redundancy.
- Number of Blocks Pending Deletion
    - The total number of blocks to be invalidated.
- Block Deletion Start Time
    - The timestamp of bock deletion start.

## NameNode Journal Status
```
Current transaction ID: 278965

Journal Manager	: FileJournalManager(root=/.../hdfs/name)
State : EditLogFileOutputStream(/.../hdfs/name/current/edits_inprogress_0000000000000000001)
```
- Current transaction ID
    - The last transaction ID that was either loaded from an image or loaded by loading edits files. Note that this is not precise value and can only be used by metrics.
- Journal Manager
    - The class of Jourmnal Manager
    - A [JournalManager](https://github.com/apache/hadoop/blob/fac9f91b2944cee641049fffcafa6b65e0cf68f2/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/namenode/JournalManager.java) is responsible for managing a single place of storing edit logs. It may correspond to multiple files, a backup node, etc. Even when the actual underlying storage is rolled, or failed and restored, each conceptual place of storage corresponds to exactly one instance of this class, which is created when the EditLog is first opened.
- State:
    - A short text snippet suitable for describing the current status of the EditLogOutputStream(EditLogOutputStream is for supporting journaling of edits logs into a persistent storage).

## NameNode Storage
```
NameNode Storage
Storage Directory : /.../hdfs/name
Type : IMAGE_AND_EDITS
State : Active
```
- Storage Directory
    - The storage directory for namenode
- Type
    - StorageDirType specific to namenode storage. A Storage directory could be of type IMAGE which stores only fsimage, or of type EDITS which stores edits or of type IMAGE_AND_EDITS which stores both fsimage and edits.
- State
    - Status of namenode.

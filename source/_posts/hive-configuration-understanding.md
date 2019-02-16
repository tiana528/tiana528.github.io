---
title: hive configuration understanding
date: 2018-11-19 21:24:42
tags:
    - hive
    - hadoop
categories:
    - big data
---
This article aims at introducing what are the manually configured settings that override the default during using hive.

## environment
This article is based on hive 2.3, hadoop 2.7, running hive on mapreduce.

## background
Hive and hadoop have many configurations and it is not striaightforward to know what configurations need to be override manually. This article aims at listing up all such configurations that we can special take care of.

## configuration reference
There are the links of the latest explanations of the configurations in the official wiki . 
- hadoop
    - [core-default.xml](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/core-default.xml)
    - [hdfs-default.xml](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)
    - [mapred-default.xml](https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml)
    - [yarn-default.xml](https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-common/yarn-default.xml)
- hive
    - [hive-site.xml](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties#ConfigurationProperties)

## overridden configurations
These are the configurations that need to take special care and be configured manually to override the default value.

- Basic settings
    - purpose
        - There are some basic settings.
    - configuration
        - fs.defaultFS
            - default : file:///
            - configured :  hdfs://<namenode_ip>:8020
            - explain : This configures the default file system. By default, it is local file system, change it to use distributed file system.
        - mapreduce.framework.name
            - default : local
            - configured : yarn
            - explain : The runtime framework for executing MapReduce jobs. Can be one of local, classic or yarn.
        - yarn.resourcemanager.hostname 
            - default:  0.0.0.0
            - configured: <the real ip>
            - explain: The hostname of the RM.

- Enable yarn log aggregation
    - purpose 
        - By default, yanr log aggregation is disabled, and yarn logs(container logs) are saved in the local of the instances where the containers run. In order to check the log, it is needed to first check the instance ip of a container, then login the instance and open the corresponding log file. When yarn log aggregation is enabled, the container's log will be aggregated to some configured place(HDFS, s3, etc.), and it is able to check the yarn log through the yarn UI, which becomes much simpler.
    - configuration 
        - yarn.log-aggregation-enable
            - default : false
            - configured : true
            - explain : This enables the yarn log aggregation.
        - yarn.nodemanager.remote-app-log-dir
            - default : /tmp/logs for hdfs
            - configured : e.g. s3a://... for s3
            - explain : This specifies the prefix of the actual path of the aggregated logs. You may want to include the cluster name in the path.
        - yarn.log-aggregation.retain-seconds
            - default : -1
            - explain : This specifies how long the aggregated logs will be kept. The default value -1 indicates keeping the logs forever. Too big value may cause a waste of disk resource, too small may make you cannot find logs when investigate recent issues. 2 weeks or so may be helpful.
        - yarn.nodemanager.log-aggregation.compression-type
            - default: none
            - configured: gz
            - explain: If, and how, aggregated logs should be compressed.
        - mapreduce.job.userlog.retain.hours
            - default : 24
            - explain : This specifies the maximum time in hours that the local yarn logs will be retained after job completion. You may want to adjust this setting together.
        - mapreduce.jobhistory.max-age-ms
            - default : 604800000
            - explain : Job history files older than this many milliseconds will be deleted when the history cleaner runs. Defaults to 604800000 (1 week). We can make it 2 weeks.
    - [helpful links](https://renenyffenegger.ch/notes/development/Apache/Hadoop/Ycccc-=ARN/log-aggregation)
- Clean HDFS trash regularly
    - purpose
        - By default, the trash folder is not cleaned up automatically. We can configure to make it be cleaned up regularly to reduce disk cost.
    - configuration
        - fs.trash.interval	
            - default : 0
            - configured : 360
            - explain : will be deleted after 6 hours
- Enable hive client authorization
    - purpose
        - By default, hive client authorization is disabled. Enabling hive client authorization can help prevent users from doing operations they are not supposed to do.
    - configuration
        - hive.security.authorization.enabled
            - default : false
            - configured : true
            - explain : Enables the hive client authorization.
        - There are also other settings as needed. I will leave it as TODO.
    - helpful links
        - [SQL Standard Based Hive Authorization](https://cwiki.apache.org/confluence/display/Hive/SQL+Standard+Based+Hive+Authorization)
- Enable HDFS compression
    - configuration
        - io.compression.codecs
            - default : 
            - configured :  org.apache.hadoop.io.compress.DefaultCodec,org.apache.hadoop.io.compress.GzipCodec,org.apache.hadoop.io.compress.BZip2Codec,org.apache.hadoop.io.compress.DeflateCodec,org.apache.hadoop.io.compress.SnappyCodec
            - explain : A comma-separated list of the compression codec classes that can be used for compression/decompression which precedence of others loaded by from the classpath.
        - mapreduce.map.output.compress
            - default : false
            - configured : true
            - explain : Compress the output of map before sending accross the network.
        - mapreduce.map.output.compress.codec
            - default : org.apache.hadoop.io.compress.DefaultCodec
            - configured : org.apache.hadoop.io.compress.SnappyCodec or org.apache.hadoop.io.compress.GzipCodec
            - explain : If the map outputs are compressed, how should they be compressed?
        - mapreduce.output.fileoutputformat.compress.type
            - default : RECORD
            - configured : BLOCK
            - explain : If the job outputs are to compressed as SequenceFiles, how should they be compressed?
- Fail jobs when exceed disk usage
    - configuration
        - mapreduce.job.dfs.storage.capacity.kill-limit-exceed
            - default : true
            - configured : false
- dfs related
    - configuration
        - dfs.namenode.avoid.read.stale.datanode
            - default : false
            - configured : true
        - dfs.namenode.avoid.write.stale.datanode
            - default : false
            - configured : true
- save job result in s3(option)
    - configuration
        - mapreduce.jobhistory.done-dir 
            - default: ${yarn.app.mapreduce.am.staging-dir}/history/done 
            - configured: s3 path
- speculative configs
    - configuration
        - mapreduce.map.speculative 
            - default: true
            - configured: false
        - mapreduce.reduce.speculative
            - default: true
            - configured: false
- enable namenode restart
    - configuration
        - yarn.nodemanager.recovery.enabled
            - default: false
            - configured: true
        - yarn.nodemanager.address
            - default:  ${yarn.nodemanager.hostname}:0
            - configured: 0.0.0.0:45454
            - explain: Default value makes namenode use different port before and after a restart, and the running clients will be broken after the restart, explicitly setting the port can solve the issue.
    - [helpful links](https://hadoop.apache.org/docs/r2.8.0/hadoop-yarn/hadoop-yarn-site/NodeManager.html)
- enable resource manager restart
    - configuration
        - yarn.resourcemanager.recovery.enabled
            - default: false
            - configured: true
    - [helpful link](https://hadoop.apache.org/docs/r2.7.5/hadoop-yarn/hadoop-yarn-site/ResourceManagerRestart.html)
- hive configs
    - configuration
        - hive.auto.convert.join
            - configured: false
        - hive.auto.convert.join.noconditionaltask
            - configured: false
        - hive.exec.compress.intermediate
            - configured: true
        - hive.exec.parallel
            - configured: true
        - hive.fetch.task.conversion
            - default: more
            - configured: none
        - hive.groupby.orderby.position.alias
            - configured: true
        - hive.log.explain.output
            - configured: true
        - hive.mapred.reduce.tasks.speculative.execution
            - configured: false
        - hive.optimize.reducededuplication
            - configured: false
        - hive.resultset.use.unique.column.names
            - configured: false
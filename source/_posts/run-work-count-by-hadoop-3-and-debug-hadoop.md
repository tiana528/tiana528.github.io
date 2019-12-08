---
title: run work count by hadoop 3 and debug hadoop by eclipse in mac
date: 2019-12-07 23:11:03
tags:
---

# description
In the previous [blog](https://mingyue.me/2019/12/03/build-hadoop-trunck-branch-and-import-into-eclipse/), we discussed about how to compile hadoop 3 from the source code and import source code into Eclipse.

As one step of the build(mvn clean install -Pdist -DskipTests -Dtar), we generated the hadoop distribution of hadoop-dist/target/hadoop-3.3.0-SNAPSHOT.tar.gz.

In this blog, we will show how to use the generated hadoop distribution to run hadoop service in mac, and how to debug hadoop client as well as service by eclipse.

hadoop version is 3.3.0-SNAPSHOT built from trunk branch.

# environment setup

## copy and extract
- cp hadoop-dist/target/hadoop-3.3.0-SNAPSHOT.tar.gz /Users/wang.yan/public_work/hadoop_distribute
- cd /Users/wang.yan/public_work/hadoop_distribute
- tar zxvf /Users/wang.yan/public_work/hadoop_distribute/hadoop-3.3.0-SNAPSHOT.tar.gz

## configure environment variable
- add HADOOP_HOME to the ~/.profile
    ```
export HADOOP_HOME=/Users/wang.yan/public_work/hadoop_distribute/hadoop-3.3.0-SNAPSHOT/
PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
    ```
- source ~/.profile

## try hadoop command
- Execute ./bin/hadoop, then following output will be generated
    ```
Usage: hadoop [OPTIONS] SUBCOMMAND [SUBCOMMAND OPTIONS]
 or    hadoop [OPTIONS] CLASSNAME [CLASSNAME OPTIONS]
  where CLASSNAME is a user-provided Java class

  OPTIONS is none or any of:

--config dir                     Hadoop config directory
--debug                          turn on shell script debug mode
--help                           usage information
buildpaths                       attempt to add class files from build tree
hostnames list[,of,host,names]   hosts to use in slave mode
hosts filename                   list of hosts to use in slave mode
loglevel level                   set the log4j level for this command
workers                          turn on worker mode
    ```

## create local paths for namenode and datanode to use
- mkdir -p /Users/wang.yan/tmp/namenode/
- mkdir -p /Users/wang.yan/tmp/datanode/

## configure hadoop properties
Change properties files under etc/hadoop/, for running hadoop in the `Pseudo-Distributed` mode.

- core-site.xml
    ```
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>
</configuration>
    ```
- hdfs-site.xml
    ```
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
      <name>dfs.name.dir</name>
      <value>file:///Users/wang.yan/tmp/namenode </value>
   </property>
   <property>
      <name>dfs.data.dir</name>
      <value>file:///Users/wang.yan/tmp/datenode </value >
   </property>
</configuration>
    ```
- mapred-site.xml
    ```
<configuration>
   <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
   </property>
    <property>
      <name>yarn.app.mapreduce.am.env</name>
      <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
      <name>mapreduce.map.env</name>
      <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
      <name>mapreduce.reduce.env</name>
      <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
</configuration>
    ```
- yarn-site.xml
    ```
<configuration>
   <property>
      <name>yarn.nodemanager.aux-services</name>
      <value>mapreduce_shuffle</value>
   </property>
</configuration>
    ```

## setup passphraseless ssh
Refer [hadoop wiki](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html#Setup_passphraseless_ssh) for this step.

I tried but it does not work properly in my local, so I skipped this step.

Because I skipped this step, I cannot use the commands of sbin/, I will use the commands of bin/ directly instead.

## cleanup before start all service
```
./bin/hdfs --daemon stop namenode
./bin/hdfs --daemon stop datanode
./bin/yarn --daemon stop resourcemanager
./bin/yarn --daemon stop nodemanager
rm -rf /Users/wang.yan/tmp/datenode/*
./bin/hdfs namenode -format
```
Note that whenever we shutdown namenode and start it, it is a new cluster, and we need to cleanup namenode and datanode data folder before start the namenode/datanode service.

## start all service
./bin/hdfs --daemon start namenode
./bin/hdfs --daemon start datanode
./bin/yarn --daemon start resourcemanager
./bin/yarn --daemon start nodemanager

## check UIs
We are able to check the UIs.
- hdfs UI : http://localhost:9870/
- resource manager UI : http://localhost:8088/

## check logs
The logs are under logs/

## test to run word count in pseudo-distributed mode
- prepare data
```
   bin/hdfs dfs -rm -r /tmp/input
   bin/hdfs dfs -rm -r /tmp/output
   bin/hdfs dfs -mkdir -p /tmp/input
   bin/hdfs dfs -ls  /tmp/input
   bin/hdfs dfs -put etc/hadoop/hadoop-env.sh   hdfs:///tmp/input/
```
- run word count hadoop job
    ```
   bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.0-SNAPSHOT.jar wordcount hdfs:///tmp/input hdfs:///tmp/output
    ```
- check result
    ```
   bin/hdfs dfs -cat /tmp/output/part-r-00000
    ```
    - Part of the output is
    ```
    "AS	1
    "License");	1
    "log	1
    #	323
    ##	12
    ###	33
    ```

# run word count in standalone node and debug by eclipse

- clean up output folder and create data
    ```
   mkdir -p /tmp/input
   rm -rf /tmp/output
   cp etc/hadoop/hadoop-env.sh /tmp/input/
    ```
- open eclipse, open WordCount class, set break point on the main function, open debug configurations, add this to the program arguments : `/tmp/input /tmp/output`.
- click the debug button, then we can debug the hadoop client side, the job finishes successfully.
- note that by such way, we are not loading any specific hadoop configuration files, so resourcemanager/nodemanager is not used, namenode/datanode is not used. We can debug and see how it works in the standalone mode.
    ```
-> construct Job, set mapper, combiner, reducer, etc.
     -> Job.waitForCompletion
         -> Job.submit
         -> Job.connect
              ->Cluster.initialize : create ClientProtocolProvider which provides the protocal for job client and job tracker(resource manager) to communicate.
	-> JobSubmitter. submitJobInternal
		-> checkSpecs :  check output
		-> JobSubmissionFiles.getStagingDir : get a wording directory for the job and create it
			= file:/tmp/hadoop/mapred/staging/wang.yan449283448/.staging
		-> submitClient.getNewJobID() : generate jobID
			= job_local2038582574_0001
		-> copyAndConfigureFiles() : upload libjars and so on to the job working directory
		-> writeSplits() : calculate split and write to the job working directory
		-> writeConf() : write configuration file to the job working directory
			= /tmp/hadoop/mapred/staging/wang.yan2038582574/.staging/job_local2038582574_0001/job.xml
		-> submitClient.submitJob() : execute the job
			-> LocalJobRunner.Job : we are not using yarn, so it is still standalone mode, using LocalJobRunner to do the work. It loads job files from the job working directory.
				-> get local job dir : file:/Users/wang.yan/public_work/hadoop/hadoop-mapreduce-project/hadoop-mapreduce-examples/build/test/mapred/local/localRunner/wang.yan/job_local470153672_0001
				-> write configuration file : /Users/wang.yan/public_work/hadoop/hadoop-mapreduce-project/hadoop-mapreduce-examples/build/test/mapred/local/localRunner/wang.yan/job_local470153672_0001/job_local470153672_0001.xml
				-> run() : use a thread to execute the map/reduce task attempt
					-> MapTask.run() : 
						-> Mapper.run() : loop input and invoke mapper
							-> TokenizerMapper.TokenizerMapper.map : the map class configured in the client side. 
    ```


# run word count in pseudo-distributed mode and debug hadoop client by eclipse

- cleanup output folder before running word count
```
   bin/hdfs dfs -rm -r /tmp/output
```
- modify etc/hadoop/hadoop-env.sh file and add following line
    ```
HADOOP_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5000 $HADOOP_OPTS"
    ```
- run word count job
    ```
   bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.0-SNAPSHOT.jar wordcount hdfs:///tmp/input hdfs:///tmp/output
    ```
    - Then this command should get stuck waiting for connection
- open eclipse -> Project -> Debug configuration -> Remote Java Applicationa -> input following and debug
    ```
    host : localhost
    port : 5000
    ```
- since we have configured hadoop configurations to run in pseudo-distributed mode, now we can debug in eclipse to know how it works for hadoop-client.

# run word count in pseudo-distributed mode and debug hadoop service by eclipse

- recover the value of hadoop-env.sh
- stop resource manager : ./bin/yarn --daemon stop resourcemanager
- modify etc/hadoop/hadoop-env.sh file and add following line
    ```
HADOOP_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5000 $HADOOP_OPTS"
    ```
- start resource manager : ./bin/yarn --daemon start resourcemanager
- eclipe -> open ResourceManager class, set breakpoint on the main function
- open eclipse -> Project -> Debug configuration -> Remote Java Applicationa -> input following and debug resource manager
    ```
    host : localhost
    port : 5000
    ```

# run word count in pseudo-distributed mode and debug hadoop child process bu eclipse

- modifying etc/hadoop/mapred-site.xml
    ```
<property>
  <name>mapred.child.java.opts</name>
  <value>-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5002</value>
</property>
    ```
- eclipse -> open YarnChild -> set break point on the main function
- open eclipse -> Project -> Debug configuration -> Remote Java Applicationa -> input following and debug the child process
    ```
    host : localhost
    port : 5002
    ```



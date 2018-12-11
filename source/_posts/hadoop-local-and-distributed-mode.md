---
title: hadoop local mode and distributed mode
date: 2018-12-09 11:55:57
tags:
    - hive
    - hadoop
categories:
    - big data
---
Whether a job runs in local mode or distributed mode is decided by *mapreduce.framework.name*. In local mode, the mapper and reducer will run locally in the same JVM with the client. In distributed mode, the job will be submitted to the resource manager. This article aims at digging into the related source code.

## Flow of execution
- Create JobClient
    - Create *JobClient* which is primary interact with the cluster.
    - During JobClient initialization, create *Cluster* which provides access to the cluster.
```java
org.apache.hadoop.mapred.JobClient
   public void init(JobConf conf) throws IOException {
       ...    
       cluster = new Cluster(conf);
       ...  
   }
```
    - During Cluster initialization, use loaded ClientProtocalProvider list to create ClientProtocol until get one, ClientProtocal is the protocal for client and central jobtracker to communicate.
```java
org.apache.hadoop.mapreduce.Cluster
  private void initialize() {
         ...
    for (ClientProtocolProvider provider : providerList) {
        ClientProtocol clientProtocol = null;
        clientProtocol = provider.create(..., conf);
        if (clientProtocol != null) {
            break;
        }
        ...
   }
```
    - There are two kinds of ClientProtocalProvider, one is LocalClientProtocolProvider for local, the other is YarnClientProtocolProvider for communicating with yarn.
        - If *mapreduce.framework.name* is yarn, then create YarnRunner.
```java
org.apache.hadoop.mapred.YarnClientProtocolProvider
  public ClientProtocol create(Configuration conf) throws IOException {
    if (MRConfig.YARN_FRAMEWORK_NAME.equals(conf.get(MRConfig.FRAMEWORK_NAME))) {
      return new YARNRunner(conf);
    }
    return null;
  }
```
        - If *mapreduce.framework.name* is local, then create LocalJobRunner, and set map to be 1.
```java
org.apache.hadoop.mapred.LocalClientProtocolProvider
  public ClientProtocol create(Configuration conf) throws IOException {
    String framework =
        conf.get(MRConfig.FRAMEWORK_NAME, MRConfig.LOCAL_FRAMEWORK_NAME);
    if (!MRConfig.LOCAL_FRAMEWORK_NAME.equals(framework)) {
      return null;
    }
    conf.setInt(JobContext.NUM_MAPS, 1);
    return new LocalJobRunner(conf);
  }
```
- Submit job through JobClient
    - Get new JobId
        - For local execution, generate local new jobId using LocalJobRunner
```java
org.apache.hadoop.mapred.LocalJobRunner
   public synchronized org.apache.hadoop.mapreduce.JobID getNewJobID() {
     return new org.apache.hadoop.mapreduce.JobID("local" + randid, ++jobid);
   }
```
        - For distributed execution, get new JobId from resource manager through YarnRunner
```java
org.apache.hadoop.mapred.YARNRunner
  public JobID getNewJobID() throws IOException, InterruptedException {
    return resMgrDelegate.getNewJobID();
  }
```
    - Actual submit job
        - For local execution
            - Create a Job object. Job extends Thread, and can be run as a Thread.
```java
org.apache.hadoop.mapred.LocalJobRunner
   public org.apache.hadoop.mapreduce.JobStatus submitJob(...) throws IOException {
      Job job = new Job(JobID.downgrade(jobid), jobSubmitDir);
      ...
   }
```
            - At the end of the constructor of Job, it starts itself as a Thread.
```java
org.apache.hadoop.mapred.LocalJobRunner
    public Job(JobID jobid, String jobSubmitDir) throws IOException {
        ...
        this.start();
        ...
    }
```
        - For distributed execution, submit the job to resource manager.
```
org.apache.hadoop.mapred.YARNRunner
    public JobStatus submitJob(JobID jobId, String jobSubmitDir, Credentials ts){
        ...
        ApplicationId applicationId =
          resMgrDelegate.submitApplication(appContext);
        ...
    }
```

## Conclusion
We can see that, whether a job runs in local mode or distributed mode is decided by *mapreduce.framework.name*. In local mode, the mapper and reducer will run locally in the same JVM with the client. In distributed mode, the job will be submitted to the resource manager.




















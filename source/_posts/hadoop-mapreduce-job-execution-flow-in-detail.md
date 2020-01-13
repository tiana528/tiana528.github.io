---
title: hadoop mapreduce job execution flow in detail
date: 2019-12-14 09:11:29
tags:
---

# Description
In this blog, we will look into the execution flow of hadoop mapreduce job (word count) in detail.

Word count job is simple and straightforward, so it is an good example to show how hadoop is working internally.

We will try to go through the whole lifecycle of the jobs, see how components are interacting by looking into the source codes.

We mainly look into client side logic, resource manager, node manager, application master, and do not look much into the internal of datanode or namenode.

Hadoop source code version is based on 3.3.0-SNAPSHOT, which is the latest at this point.

# Understand hadoop RPC
Considering RPC is heavily used in hadoop, so understanding it is important for reading source codes.

There are already many articles written by other people about the hadoop RPC, I will just show one graph about it.

![](hadoop_rpc_example.jpg)

This graph shows the class relationship around namenode client-server RPC call.

As we can see, the protobuffer specification is written in ClientNamenodeProtocal.proto file.

ClientNamenodeProtocal.java is an interface generated from the proto file.

All other classes(in client and server side), contains the functions provided by the ClientNamnodeProtocal interface, either by extends relationship or by associate relationship.

One an api call in invoked, data flow is : ClientNamenodeProtocalTranslatorPB(client side) -> ClientNamenodeProtocalServerSideTranslatorPB(server side) -> NamenodeRpcServer


# Job execution flow
## Job submission

```
WordCount(hadoop-mapreduce-examples)
-> Job.waitForCompletion  (hadoop-mapreduce-client-core)
	-> JobSubmitter.submitJobInternal (hadoop-mapreduce-client-core)
		-> submitClient.getNewJobID (hadoop-mapreduce-client-core)
			-> YARNRunner.getNewJobID (hadoop-mapreduce-client-jobclient)
				-> ResourceMgrDelegate.getNewJobID (hadoop-mapreduce-client-jobclient)
					-> YarnClientImpl.createApplication() (hadoop-yarn-client)
						-> get ApplicationSubmissionContext which is actually ApplicationSubmissionContextPBImpl
						-> get GetNewApplicationRequest which is actually GetNewApplicationRequestPBImpl
						-> has a field rmClient which is a proxy (ApplicationClientProtocol) , created by `ClientRMProxy.createRMProxy`
						-> rmClient.getNewApplication 
							-> ApplicationClientProtocolPBClientImpl.getNewApplication (hadoop-yarn-common)
								=> ApplicationClientProtocolPBServiceImpl.getNewApplication (hadoop-yarn-common)
									=> ClientRMService.getNewApplication (hadoop-yarn-server-resourcemanager)
									=> generate real ApplicationId which is incremental
		-> copyAndConfigureFiles
			-> JobResourceUploader.uploadResources (hadoop-mapreduce-client-core)
				-> upload resource that are added by the command line , tmpfiles, tmpjars ,tmparchives, jar file , log4j property file
		-> writeSplits
			-> get InputFormat through getInputFormatClass, then invoke InputFormat.getSplits to calculate splits, then upload split info to the staging directory
                -> As for word count, FileInputFormat.getSplits() is invoked
		-> upload job.xml configs 
		-> submitClient.submitJob
			-> YarnRunner.submitJob
				-> ResourceMgrDelegate.submitApplication
				-> ResourceMgrDelegate.getApplicationReport
					... =>ClientRMService.submitApplication (hadoop-yarn-server-resourcemanager)
```
## Resource manager application lifecycle : RMAppState NEW -> NEW_SAVING
```
State machine : RMAppImpl.StateMachineFactory(hadoop-yarn-server-resourcemanager)
RMStateStore : handles RMStateStoreAppEvent event
RMAppImpl : handles RMAppEventType event


ClientRMService.submitApplication (hadoop-yarn-server-resourcemanager)
	=> rmContext getRMApps : all apps that have been put into rmContext
	=> RMAppManager.submitApplication
		=> createAndPopulateNewRMApp : create RMApp, put into rmContext.getRMApps
		=> this.rmContext.getDispatcher().getEventHandler().handle(new RMAppEvent(applicationId, RMAppEventType.START));
		=> trigger the event of RMAppEventType.START
			... => AsyncDispatcher is the dispatcher (hadoop-yarn-common)
					=> eventHandler is GenericEventHandler, the event is put into a queue, and AsyncDispatcher has a thread to loop checking all queued events, and dispatch the events(void dispatch(Event event) ).
						=> In ResourceManager, ApplicationEventDispatcher is registered to handle event type of RMAppEventType, so the RMAppEventType.START event will be dispatched to ApplicationEventDispatcher.
							=> RMAppImpl.handle() handles the event
							=> the state machine factory(StateMachineFactory) is also initialized in the ResourceManager, the initial state is NEW.
							=> We can see one registered transaction is : addTransition(RMAppState.NEW, RMAppState.NEW_SAVING,  RMAppEventType.START, new RMAppNewlySavingTransition()). It means if the current state is NEW, and RMAppEventType.START event happens, then invoke RMAppNewlySavingTransition and convert the state to NEW_SAVING.

## RMAppState NEW_SAVING -> APP_NEW_SAVED

At the end of RMAppNewlySavingTransition, RMStateStore.storeNewApplication
    => stores application state, and triggers RMStateStoreAppEvent event, event type is RMStateStoreEventType.STORE_APP
        => RMStateStore.StoreAppTransition handles the event
            => LeveldbRMStateStore.storeApplicationStateInternal : (if leveldb is configured to be used) saves the appid->appdata mapping to leveldb.
            => trigger event RMAppEventType.APP_NEW_SAVED


## RMAppState APP_NEW_SAVED -> SUBMITTED
RMAppImpl.AddApplicationToSchedulerTransition handles the APP_NEW_SAVED type event
    => dispatch AppAddedSchedulerEvent event
        => CapacityScheduler.handle (yarn.resourcemanager.scheduler.class)
            => CapacityScheduler.addApplication
                => CSQueue.submitApplication : Submit a new application to the queue
                => Put the application into a field of : applications
                => dispatch RMAppEventType.APP_ACCEPTED event
    => if timeline server is enabled (yarn.timeline-service.generic-application-history.enabled)
        => dispatch WritingApplicationStartEvent event, event type is WritingHistoryEventType.APP_START
            => RMApplicationHistoryWriter.handleWritingApplicationHistoryEvent
                => ApplicationHistoryWriter.applicationStarted
                    => FileSystemApplicationHistoryStore.applicationStarted (yarn.timeline-service.generic-application-history.store-class)
                        => working path : yarn.timeline-service.generic-application-history.fs-history-store.uri or "\${hadoop.tmp.dir"}/yarn/timeline/generic-history"
                        => root dir : working path + ApplicationHistoryDataRoot
                        => application history file : root dir + application id (This is the file in the local file system of resource manager for saving one application history. )
                        => write ApplicationStartDataPBImpl to the application history file
        => if system publisher is enabled(yarn.system-metrics-publisher.enabled) and timeline server v1 is enabled
            => TimelineServiceV1Publisher.appCreated
                => publish application name, application type, user, tags, priority, and so on to timelineserver.
                    ... => TimelineServiceV1Publisher.putEntity ...
                            => Go to the timelineserver world

```
## RMAppState SUBMITTED -> ACCEPTED
```
RMAppImpl.StartAppAttemptTransition
    => RMAppImpl.createAndStartNewAttempt
        => create new application attempt = appId + attemptId(increase by 1 : 1, 2, 3...)
        => create RMAppAttemptImpl, put to attempts field, update the currentAttempt
        => trigger RMAppStartAttemptEvent event with the attempt id, event type : RMAppAttemptEventType.START
```
## RMAppAttemptState NEW -> SUBMITTED
```
RMAppAttemptImpl.AttemptStartedTransition handles RMAppAttemptEventType.START : RMAppAttemptState NEW -> SUBMITTED
    => appAttempt.masterService.registerAppAttempt : Register with the ApplicationMasterService
         => Note that ApplicationMasterService is created in the ResourceManager
         => ApplicationMasterService.registerAppAttempt
         => trigger AppAttemptAddedSchedulerEvent event(SchedulerEventType.APP_ATTEMPT_ADDED)
             => CapacityScheduler.handle deal with the above event
                 => CapacityScheduler.addApplicationAttempt
                     => get SchedulerApplication from applications fields, create new FiCaSchedulerApp(for the attempt), set the attempt to the application as the current app attempt.
                     => queue.submitApplicationAttempt
                         => LeafQueue.submitApplicationAttempt, put into applicationAttemptMap
                         => use yarn.scheduler.capacity.resource-calculator(or DefaultResourceCalculator) as calculator to calculate whether resource is enough
                             => if resource is enough(lastClusterResource is larger than 0), activateApplications
                                 => configure the queue(internal datastructure) for the resource of application master
                             => if resource is not enough, do not activeApplication now
                     => dispatch RMAppAttemptEventType.ATTEMPT_ADDED event
```
## RMAppAttemptState SUBMITTED -> SCHEDULED
```
ScheduleTransition handles the RMAppAttemptEventType.ATTEMPT_ADDED event
    => allocate container for the application master
        => CapacityScheduler.allocate (A view of the resources that will be allocated from this application)
            => create an Allocation object, with the request content inside
```

## RMAppAttemptState SCHEDULED -> ALLOCATED_SAVING
```
This is not very straightforward.

(NM) When NodeManager service starts, NodeManager create NodeStatusUpdater service
(NM)     -> NodeStatusUpdater.registerWithRM
(NM)          -> ResourceTracker.registerNodeManager, get node id, host, port, capacity
(RM)                -> create RMNodeImpl(Node managers information on available resources)
(...)                   -> Dispatch RMNodeEventType.STARTED event
                            -> AddNodeTransition handles the event, NodeState changed from NEW to RUNNING
                                -> dispatch SchedulerEventType.NODE_ADDED event and NodesListManagerEventType.NODE_USABLE event
                                    -> CapacityScheduler handles SchedulerEventType.NODE_ADDED event, create FiCaSchedulerNode


(NN) NodeStatusUpdateImpl.StatusUpdaterRunnable send heartbeat to resourcemanager regularly
(RM)    -> NodeManager.heartbeat
            -> ResourceTrackerService.nodeHeartbeat
                -> dispatch RMNodeEventType.STATUS_UPDATE event
                    -> StatusUpdateWhenHealthyTransition handles the event
                        -> dispatch SchedulerEventType.NODE_UPDATE event
                            -> CapacityScheduler handles the event, invoke CapacityScheduler.nodeUpdate(), try to do scheduling
                                -> CapacityScheduler.allocateContainersToNode
                                    -> CapacityScheduler.allocateContainerOnSingleNode
                                        -> CapacityScheduler.allocateOrReserveNewContainers
                                            -> LeafQueue.assignContainers
                                                -> FiCaSchedulerApp.assignContainerys
                                                    -> ContainerAllocator.assignContainers
                                                        -> RegularContainerAllocator.assignContainers
                                                            -> RegularContainerAllocator.allocate, it invokes tryAllocateOnNode which actually allocate the resource on node
                                                                -> RegularContainerAllocator.doAllocation
                                                                    -> RegularContainerAllocator.handleNewContainerAllocation
                                                                        -> FiCaSchedulerApp.allocate
                                                                            -> creat RMContainerImpl(Represents the ResourceManager's view of an application container.)

                                            -> CapacityScheduler.submitResourceCommitRequest
                                                -> CapacityScheduler.tryCommit
                                                    -> CapacityScheduler.submitResourceCommitRequest
                                                        -> FiCaSchedulerApp.apply
                                                            -> dispatch event RMContainerEventType.START
                                                                -> ContainerStartedTransition handles the event, and RMContainerEventType NEW->ALLOCATED
                                                                    -> dispatch RMAppAttemptEventType.CONTAINER_ALLOCATED event
                                                                        -> RMAppAttemptImpl.AMContainerAllocatedTransition handles the event and RMAppAttemptState SCHEDULED->ALLOCATED_SAVING
                                                                            -> set master container, RMContainerImpl.setAMContainer(true), trigger RMStateStoreEventType.STORE_APP_ATTEMPT event to store the state of RM
                                                                                -> StoreAppAttemptTransition handles the event, saves the state to leveldb(for example), then dispatch event RMAppAttemptEventType.ATTEMPT_NEW_SAVED

```


## RMAppAttemptState ALLOCATED_SAVING -> ALLOCATED
```
AttemptStoredTransition handles the RMAppAttemptEventType.ATTEMPT_NEW_SAVED event
    -> RMAppAttemptImpl.launchAttempt, trigger AMLauncherEventType.LAUNCH event
        -> ApplicationMasterLauncher.launch
            -> Use ContainerManagementProtocol protocal to interact with NodeManager, connect nodemanager
            -> Create ContainerLaunchContext(represents all of the information needed by the {@code NodeManager} to launch a container.)
            -> Send StartContainerRequest request to NodeManager and get response.
                (NM) -> ContainerManagerImpl.startContainers
                    (NM) -> get ContainerLaunchContext, create ContainerImpl, create ApplicationImpl, dispatch ApplicationInitEvent
                        (NM) -> ...... -> DefaultContainerExecutor.launchContainer
                                            -> exec /bin/bash -c "JAVA_HOME/bin/java org.apache.hadoop.mapreduce.v2.app.MRAppMaster"
                                            -> Then application master service process is created
            -> dispatch RMAppAttemptEventType.LAUNCHED event
```
## RMAppAttemptState ALLOCATED -> LAUNCHED
```
AMLaunchedTransition handles the RMAppAttemptEventType.LAUNCHED event
    -> register RMAppEventType.ATTEMPT_LAUNCHED event
```
## RMAppState ACCEPTED -> ACCEPTED
```
AttemptLaunchedTransition (update the launchTime and publish to ATS)
    -> dispatch RMStateStoreEventType.UPDATE_APP event
        -> UpdateAppTransition handles RMStateStoreEventType.UPDATE_APP event
            -> store the state and dispatch APP_UPDATE_SAVED event
```
## RMAppAttemptState LAUNCHED -> RUNNING, RMAppState ACCETPTED -> RUNNING
```
(AM) Application master (MRAppMaster) start
    -> (AM) ContainerAllocatorRouter service is created to handle container allocation
        -> (AM) if uber mode, use LocalContainerAllocator
        -> (AM) if not uber mode, use RMContainerAllocator, note that RMContainerAllocator is also RMCommunicator
            -> (AM) RMCommunicator.serviceStart
                -> (AM) RMCommunicator.register : registerApplicationMaster , this will communicate with the Resource Manager.
                    -> (RM) DefaultAMSProcessor.registerApplicationMaster : resource manager side handles the request
                        -> (RM) dispatch RMAppAttemptEventType.REGISTERED event
                            -> (RM) AMRegisteredTransition handles the event, and RMAppAttemptState LAUNCHED -> RUNNING
                                -> (RM) dispatch RMAppEventType.ATTEMPT_REGISTERED event, WritingHistoryEventType.APP_ATTEMPT_START event
```
## Appliation master allocate resource from resource manager, start container by communicating with node manager, run map reduce tasks in the container
```
(AM)
Inside RMCommunicator.serviceStart, it creates a thread (AllocatorRunnable) for allocating containers for every 1 second.
    -> RMContainerAllocator.heartbeat
        -> RMContainerAllocator.getResources
            -> RMContainerAllocator.makeRemoteRequest
                -> ... ApplicationMasterProtocolPBClientImpl.allocate()
                    -> (RM) ApplicationMasterService.allocate and response
        -> (AM) loop all assigned containers, check memory requirements, blacklists, and so on, then assign the container : RMContainerAllocator.assignContainers
            -> dispatch TaskAttemptContainerAssignedEvent event (assign container X to task Y on node Z)
                -> TaskAttemptImpl.ContainerAssignedTransition handles the event, state : TaskAttemptStateInternal.ASSIGNED => TaskAttemptEventType.TA_ASSIGNED
                    -> create ContainerLaunchContext, setup commands for creating the container(MapReduceChildJVM.getVMCommand): /bin/java ... YarnChild
                    -> dispatch ContainerRemoteLaunchEvent event
                        -> ContainerLauncherImpl.launch, connect nodemanager, start actual container
                           -> org.apache.hadoop.mapred.YarnChild#main, args : TaskUmbilicalProtocol's host and port for providing service, task attempt's TaskAttemptID, task attemp's JVMId
                        -> dispatch TaskAttemptContainerLaunchedEvent event
                            ->LaunchedContainerTransition handles the event, TaskAttemptStateInternal.ASSIGNED => TaskAttemptStateInternal.RUNNING
                                -> dispatch TaskEventType.T_ATTEMPT_LAUNCHED event
                                    -> LaunchTransition handles the event TaskStateInternal.SCHEDULED => TaskStateInternal.RUNNING
```
## YarnChild works and perform map tasks
```
YarnChild.main
    -> read arguments : TaskUmbilicalProtocol's host and port(which is application master MRAppMaster) for providing service, task attempt's TaskAttemptID, task attemp's JVMId
        -> get map task from application master through TaskUmbilicalProtocal and run it.
            -> MapTask.run
                -> create mapper, inputFormat, read split
                -> mapper.run() : this is the actual word count mapper(WordCount.TokenizerMapper)
                    -> Context is : org.apache.hadoop.mapreduce.lib.map.WrappedMapper, which is a wrapped of MapContextImpl
                    -> write output data to NewDirectOutputCollector(reducer is 0) or NewOutputCollector(reducer is not 0)
                        -> MapOutputBuffer.collect put the result in the memory buffer as intermediate storage
                -> ... (There is a lot of logic here, we won't go into much details)
```
## Similar progress as above and reducer task starts, until job ends
```
...(omit some part)...
YarnChild.main
    -> read arguments : TaskUmbilicalProtocol's host and port(which is application master MRAppMaster) for providing service, task attempt's TaskAttemptID, task attemp's JVMId
        -> get reducer task from application master through TaskUmbilicalProtocal and run it.
            -> ReduceTask.run
                -> create reducer
                -> reducer.run() : this is the actual word count mapper(WordCount.IntSumReducer)
                -> ... (There is a lot of logic here, we won't go into much details)
                -> umbilical.done
                    -> (AM) TaskAttemptListenerImpl.done() : trigger TaskAttemptEventType.TA_DONE event
                        -> (AM) TaskAttemptImpl.MoveContainerToSucceededFinishingTransition handles the event, TaskAttemptStateInternal RUNNING -> SUCCESS_FINISHING_CONTAINER
                            -> (AM) triggers event TaskEventType.T_ATTEMPT_SUCCEEDED
                                -> (AM) TaskImpl.AttemptSucceededTransition handle the event, TaskStateInternal.RUNNING => TaskStateInternal.SUCCEEDED
                                    -> (AM) trigger JobEventType.JOB_TASK_ATTEMPT_COMPLETED event
                                        -> (AM) JobImpl.TaskAttemptCompletedEventTransition handles the event, JobStateInternal.RUNNING => JobStateInternal.RUNNING
                                        -> ...
```














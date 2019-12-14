---
title: word count hadoop job execution flow in detail
date: 2019-12-14 09:11:29
tags:
---

# Description
In this blog, we will look into the execution flow of hadoop word count job in detail.

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

# Word count execution flow
-> client side logic
=> server side logic

## Job submission

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

## Resource manager application lifecycle
State machine : RMAppImpl.StateMachineFactory(hadoop-yarn-server-resourcemanager)
RMStateStore : handles RMStateStoreAppEvent event
RMAppImpl : handles RMAppEventType event
### RMAppState NEW -> NEW_SAVING
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

### RMAppState NEW_SAVING -> APP_NEW_SAVED

At the end of RMAppNewlySavingTransition, RMStateStore.storeNewApplication
    => stores application state, and triggers RMStateStoreAppEvent event, event type is RMStateStoreEventType.STORE_APP
        => RMStateStore.StoreAppTransition handles the event
            => LeveldbRMStateStore.storeApplicationStateInternal : (if leveldb is configured to be used) saves the appid->appdata mapping to leveldb.
            => trigger event RMAppEventType.APP_NEW_SAVED


### RMAppStore APP_NEW_SAVED -> SUBMITTED
RMAppImpl.AddApplicationToSchedulerTransition handles the APP_NEW_SAVED type event
    => dispatch AppAddedSchedulerEvent event
    => if timeline server is enabled (yarn.timeline-service.generic-application-history.enabled)
        => dispatch WritingApplicationStartEvent event, event type is WritingHistoryEventType.APP_START
            => RMApplicationHistoryWriter.handleWritingApplicationHistoryEvent
                => ApplicationHistoryWriter.applicationStarted
                    => FileSystemApplicationHistoryStore.applicationStarted (yarn.timeline-service.generic-application-history.store-class)
        => if system publisher is enabled(yarn.system-metrics-publisher.enabled) and timeline server v1 is enabled
            => TimelineServiceV1Publisher.appCreated
                => publish application name, application type, user, tags, priority, and so on to timelineserver.
                    ... => TimelineServiceV1Publisher.putEntity ...
                            => Go to the timelineserver world






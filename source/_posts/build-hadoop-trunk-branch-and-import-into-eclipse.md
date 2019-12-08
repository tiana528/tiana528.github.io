---
title: build hadoop 3 trunk branch and import into eclipse in mac
date: 2019-12-03 18:50:30
tags:
---

# Description
I tried to build the trunk branch of hadoop git repository and imported it into eclipse then resolved the issues.
The date I did it is 2019/11/03, the latest commit is : 6b2d6d4aafb110bef1b77d4ccbba4350e624b57d, version is 3.3.0-SNAPSHOT.

For some necessary tools which are needed, we can refer to this to know what and which version to install :  [Building on macOS (without Docker)](https://github.com/apache/hadoop/blob/18059acb6ae16e72a6cdd08795f6281cda122bff/BUILDING.txt#L377-L410).
Alternatively, we can use `./start-build-env.sh` command to simply build the environment, I am not using it because the build time using it is much longer than build without docker.

# Steps

## download hadoop repository
- git clone https://github.com/apache/hadoop
- cd hadoop
- cat hadoop-project/pom.xml | grep proto
  - check the version of the protoc that is being used, in my case, version 3.5.1 is being used

## compile and build proto
Note that the same version is recommended to be used as the one we checked in the previous step.

I am installing it under `/usr/local/protoc`, you can install wherever you want, and even someplace in your home folder.
- git clone https://github.com/protocolbuffers/protobuf.git
- cd protobuf
- git checkout tags/v3.5.1 -b v3.5.1
- autoreconf --install
- ./configure --prefix=/usr/local/protoc
- make 
- sudo make install
- Update ~/.bashrc file to add a line of : PATH="/usr/local/protoc/bin:${PATH}"
- source ~/.bashrc

## compile hadoop
- cd hadoop
- mvn clean install -Pdist -DskipTests -Dtar
- mvn eclipse:clean eclipse:eclipse -DskipTests

## import hadoop into eclipse and solve issues
After importing all hadoop projects into eclipse, there are 400 build errors shown in the error console. I will show how to solve them manually.
Note that my way of fixing them is only aiming at resolving eclipse issues quickly, and not an ideal way, ideally everything should be resolved just after running the maven command.

### about cannot nest issue
(1) error message : hadoop-yarn-common project has build error, Cannot nest 'hadoop-yarn-common/src/test/resources/resource-types' inside 'hadoop-yarn-common/src/test/resources'. To enable the nesting exclude 'resource-types/' from 'hadoop-yarn-common/src/test/resources'

(2) solve approach : hadoop-yarn-common project -> properties -> build path -> add `resource-types/` to the exclude path

(3) remaining errors : 399

### about missing required folder issue
(1) error message : Project 'hadoop-streaming' is missing required source folder: '/Users/wang.yan/public_work/hadoop/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager/conf'

(2) solve appraoch : hadoop-streaming -> properties -> source -> remove the one has error

(3) remaining errors: 550 (the number increases lol)

###  about package-info issue
(1) error message : The type package-info is already defined	package-info.java	/hadoop-azure/src/main/java/org/apache/hadoop/fs/azurebfs/constants

(2) solve approach : hadoop-azure -> build -> resource -> src/main/java exclude : `**/package-info.java`

(3) remaining errors : 545

### about Builder cannot be resolved issue
(1) error message : Builder cannot be resolved to a type	ITUseMiniCluster.java	/hadoop-client-integration-tests/src/test/java/org/apache/hadoop/example

(2) solve approach :  hadoop-client-integration-tests -> add project dependency of hadoop-hdfs, hadoop-common, hadoop-hdfs-client

(3) remaining errors : 516

### about access restriction issue

(1) error message : The method 'Unsafe.arrayBaseOffset(Class<?>)' is not API (restriction on required library '/Library/Java/JavaVirtualMachines/jdk1.8.0_172.jdk/Contents/Home/jre/lib/rt.jar')	FastByteComparisons.java	/hadoop-common/src/main/java/org/apache/hadoop/io

(2) solve approach : Window -> Preferences -> Java -> Compiler -> Errors/Warnings -> Deprecated and restricted API -> Forbidden reference (access rules) -> Warnings

(3) remaining errors : 429

### about AvroRecord issue
(1) error message :  AvroRecord cannot be resolved to a type	TestAvroSerialization.java	/hadoop-common/src/test/java/org/apache/hadoop/io/serializer/avro

(2) solve approach : 
- Download avro-tools-x.x.x.jar jar file from the Internet
- cd hadoop-common-project/hadoop-common/src/test/avro
- java -jar ~/Downloads/avro-tools-1.7.7.jar compile schema avroRecord.avsc ../java

(3) remaining errors : 412

### about Proto
(1) error message : EchoRequestProto cannot be resolved	RPCCallBenchmark.java	/hadoop-common/src/test/java/org/apache/hadoop/ipc	line 386	Java Problem

(2) solve approach :
- cd hadoop-common-project\hadoop-common\src\test\proto
- protoc --javaout=../java *.proto

(3) remaining errors : 137

### more about proto
Do the same as above

- cd ./hadoop-yarn-project/hadoop-yarn/hadoop-yarn-client/src/test/proto/
- protoc --javaout=../java *.proto --proto_pth=.  --proto_path=/Users/wang.yan/hadoop_source/hadoop/./hadoop-yarn-project/hadoop-yarn/hadoop-yarn-api/src/main/proto/ --proto_path=/Users/wang.yan/hadoop_source/hadoop/hadoop-common-project/hadoop-common/src/main/proto

- cd ./hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-resourcemanager/src/test/proto/test_client_tokens.proto
- protoc --javaout=../java *.proto --proto_path=.  --proto_path=/Users/wang.yan/hadoop_source/hadoop/./hadoop-yarn-project/hadoop-yarn/hadoop-yarn-api/src/main/proto/ --proto_path=/Users/wang.yan/hadoop_source/hadoop/hadoop-common-project/hadoop-common/src/main/proto

- cd ./hadoop-yarn-project/hadoop-yarn/hadoop-yarn-server/hadoop-yarn-server-tests/src/test/proto/
- protoc --javaout=../java *.proto --proto_path=.  --proto_path=/Users/wang.yan/hadoop_source/hadoop/./hadoop-yarn-project/hadoop-yarn/hadoop-yarn-api/src/main/proto/ --proto_path=/Users/wang.yan/hadoop_source/hadoop/hadoop-common-project/hadoop-common/src/main/proto

Remaining errors : 0


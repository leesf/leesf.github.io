---
title: 深入理解Flink-On-Yarn模式
date: 2019-07-22 09:48:18
permalink: /tech/flink-on-yarn.html
categories:
- technique
tags:
- Flink
---

# 1. 前言

> Flink提供了两种在yarn上运行的模式，分别为Session-Cluster和Per-Job-Cluster模式，本文分析两种模式及启动流程。



下图展示了Flink-On-Yarn模式下涉及到的相关类图结构

![集群启动入口类](https://raw.githubusercontent.com/leesf/blogPhotos/master/flink/flink-on-yarn/entrypoint-class.png)

![组件类图结构](https://raw.githubusercontent.com/leesf/blogPhotos/master/flink/flink-on-yarn/component-class.png)

# 2. Session-Cluster模式

> Session-Cluster模式需要先启动集群，然后再提交作业，接着会向yarn申请一块空间后，资源永远保持不变。如果资源满了，下一个作业就无法提交，只能等到yarn中的其中一个作业执行完成后，释放了资源，下个作业才会正常提交。所有作业共享Dispatcher和ResourceManager；共享资源；适合规模小执行时间短的作业。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/flink/flink-on-yarn/yarn-session.png)

## 2.1. 启动集群

运行`bin/yarn-session.sh`即可默认启动包含一个TaskManager（内存大小为1024MB，包含一个Slot）、一个JobMaster（内存大小为1024MB），当然可以通过指定参数控制集群的资源，如**-n**指定TaskManager个数，**-s**指定每个TaskManager中Slot的个数；其他配置项，可参考

[启动yarn-session]: https://ci.apache.org/projects/flink/flink-docs-stable/ops/deployment/yarn_setup.html#start-flink-session

下面以bin/yarn-session.sh为例，分析Session-Cluster启动流程。



## 2.2. 流程分析

下面分为本地和远程分析启动流程，其中本地表示在客户端的启动流程，远端则表示通过Yarn拉起Container的流程；

### 2.2.1 本地流程

- Session启动入口为**FlinkYarnSessionCli#main**
- 根据传入的参数确定集群的资源信息（如多少个TaskManager，Slot等）
- 部署集群AbstractYarnClusterDescriptor#deploySessionCluster -> AbstractYarnClusterDescriptor#deployInternal
  - 进行资源校验（如内存大小、vcore大小、队列）
  - 通过YarnClient创建Application
  - 再次校验资源
  - AbstractYarnClusterDescriptor#startAppMaster启动AppMaster
    - 初始化文件系统（HDFS）
    - 将log4j、logback、flink-conf.yaml、jar包上传至HDFS
    - 构造AppMaster的Container（确定Container进程的入口类YarnSessionClusterEntrypoint），构造相应的Env
    - YarnClient向Yarn提交Container申请
    - 跟踪ApplicationReport状态（确定是否启动成功，可能会由于资源不够，一直等待）
  - 启动成功后将对应的ip和port写入flinkConfiguration中
  - 创建与将集群交互的ClusterClient
    - 根据flink-conf的HA配置创建对应的服务（如StandaloneHaServices、ZooKeeperHaServices等）
    - 创建基于Netty的RestClient；
    - 创建/rest_server_lock、/dispatcher_lock节点（以ZK为例）
    - 启动监听节点的变化（主备切换）
  - 通过ClusterClient获取到appId信息并写入本地临时文件

经过上述步骤，整个客户端的启动流程就结束了，下面分析yarn拉起Session集群的流程，入口类在申请Container时指定为YarnSessionClusterEntrypoint。

### 2.2.2 远端流程

* 远端宿主在Container中的集群入口为**YarnSessionClusterEntrypoint#main**

*  ClusterEntrypoint #runClusterEntrypoint -> ClusterEntrypoint#startCluster启动集群

  * 初始化文件系统
  * 初始化各种Service(如：创建RpcService（AkkaRpcService）、创建HAService、创建并启动BlobServer、创建HeartbeatServices、创建指标服务并启动、创建本地存储ExecutionGraph的Store）
  * 创建DispatcherResourceManagerComponentFactory（SessionDispatcherResourceManagerComponentFactory），用于创建DispatcherResourceManagerComponent（用于启动Dispatcher、ResourceManager、WebMonitorEndpoint）
  * 通过DispatcherResourceManagerComponentFactory创建DispatcherResourceManagerComponent
    * 创建/dispatcher_lock节点，/resource_manager_lock节点
    * 创建DispatcherGateway、ResourceManagerGateway的Retriever（用于创建RpcGateway）
    * 创建DispatcherGateway的WebMonitorEndpoint并启动
    * 创建JobManager的指标组
    * 创建ResourceManager、Dispatcher并启动进行ZK选举
    * 返回SessionDispatcherResourceManagerComponent

  经过上述步骤就完成了集群的启动；

## 2.3. 启动任务

当启动集群后，即可使用`./flink run -c mainClass /path/to/user/jar`向集群提交任务。

## 2.4 流程分析

同样，下面分为本地和远程分析启动流程，其中本地表示在客户端提交任务流程，远端则表示集群收到任务后的处理流程。

###  2.4.1 本地流程

* 程序入口为**CliFrontend#main**
* 解析处理参数
* 根据用户jar、main、程序参数、savepoint信息生成PackagedProgram
* 获取session集群信息
* 执行用户程序
  * 设置ClassLoader
  * 设置Context
  * 执行用户程序main方法（当执行用户业务逻辑代码时，会解析出StreamGraph然后通过ClusterClient#run来提交任务），其流程如下：
    * 获取任务的JobGraph
    * 通过RestClusterClient#submitJob提交任务
    * 创建本地临时文件存储JobGraph
    * 通过RestClusterClient向集群的rest接口提交任务
    * 处理请求响应结果
  * 重置Context
  * 重置ClassLoader

经过上述步骤，客户端提交任务过程就完成了，主要就是通过RestClusterClient将用户程序的JobGraph通过Rest接口提交至集群中。

### 2.4.2 远端流程

远端响应任务提交请求的是RestServerEndpoint，其包含了多个Handler，其中JobSubmitHandler用来处理任务提交的请求；

* 处理请求入口：**JobSubmitHandler#handleRequest**
* 进行相关校验
* 从文件中读取出JobGraph
* 通过BlobClient将jar及JobGraph文件上传至BlobServer中
* 通过Dispatcher#submitJob提交JobGraph
* 通过Dispatcher#runJob运行任务
  * 创建JobManagerRunner（处理leader选举）
  * 创建JobMaster（实际执行任务入口，包含在JobManagerRunner）
  * 启动JobManagerRunner（会进行leader选举，ZK目录为leader/${jobId}/job_manager_lock）
  * **当为主时会调用JobManagerRunner#grantLeadership方法**
    * 启动JobMaster
    * 将任务运行状态信息写入ZK（/${AppID}/running_job_registry/${jobId}）
    * 启动JobMaster的Endpoint
    * 开始调度任务JobMaster#startJobExecution

接下来就进行任务具体调度（构造ExecutionGraph、申请Slot等）流程，本篇文章不再展开介绍。

# 3. Per-Job-Cluster模式

> 一个任务会对应一个Job，每提交一个作业会根据自身的情况，都会单独向yarn申请资源，直到作业执行完成，一个作业的失败与否并不会影响下一个作业的正常提交和运行。独享Dispatcher和ResourceManager，按需接受资源申请；适合规模大长时间运行的作业。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/flink/flink-on-yarn/per-job.png)

## 3.1 启动任务

启动Per-Job-Cluster任务，可通过`./bin/flink run -m yarn-cluster -d -c mainClass /path/to/user/jar`命令使用分离模式启动一个集群，即单任务单集群；

## 3.2. 流程分析

与Session-Cluster类似，我们对Per-Job-Cluster模式也分为本地和远端。

### 3.2.1 本地流程

* 与Session-Cluster模式类似，入口也为CliFrontend#main
* 解析处理参数
* 根据用户jar、main、程序参数、savepoint信息生成PackagedProgram
* 根据PackagedProgram创建JobGraph（对于非分离模式还是和Session模式一样，模式Session-Cluster）
* 获取集群资源信息
* 部署集群YarnClusterDesriptor#deployJobCluster -> AbstractYarnClusterDescriptor#deployInternal；后面流程与Session-Cluster类似，值得注意的是在AbstractYarnClusterDescriptor#startAppMaster中与Session-Cluster有一个显著不同的就是其会**将任务的JobGraph上传至Hdfs供后续服务端使用**

经过上述步骤，客户端提交任务过程就完成了，主要涉及到文件（JobGraph和jar包）的上传。

### 3.2.2 远端流程

* 远端宿主在Container中的集群入口为**YarnJobClusterEntrypoint#main**
* ClusterEntrypoint#runClusterEntrypoint -> ClusterEntrypoint#startCluster启动集群
* 创建JobDispatcherResourceManagerComponentFactory（用于创建JobDispatcherResourceManagerComponent）
* 创建ResourceManager（YarnResourceManager）、Dispatcher（MiniDispatcher），其中在创建MiniDispatcher时会从之前的**JobGraph文件中读取出JobGraph**，并启动进行ZK选举
* **当为主时会调用Dispatcher#grantLeadership方法**
  * Dispatcher#recoverJobs恢复任务，获取JobGraph
  * Dispatcher#tryAcceptLeadershipAndRunJobs确认获取主并开始运行任务
    * Dispatcher#runJob开始运行任务（创建JobManagerRunner并启动进行ZK选举），后续流程与Session-Cluster相同，不再赘述

# 4. 总结

Flink提供在Yarn上两种运行模式：Session-Cluster和Per-Job-Cluster，其中Session-Cluster的资源在启动集群时就定义完成，后续所有作业的提交都共享该资源，作业可能会互相影响，因此比较适合小规模短时间运行的作业，对于Per-Job-Cluster而言，所有作业的提交都是单独的集群，作业之间的运行不受影响（可能会共享CPU计算资源），因此比较适合大规模长时间运行的作业。










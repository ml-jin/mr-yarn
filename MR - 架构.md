# MR - 架构

[TOC]

# MapReduce框架结构

Map/Reduce是一个用于大规模数据处理的分布式计算模型，它最初是由Google工程师设计并实现的，Google已经将它完整的[MapReduce](http://labs.google.com/papers/mapreduce.html)论文公开发布了。其中对它的定义是，Map/Reduce是一个编程模型（programming model），是一个用于处理和生成大规模数据集（processing and generating large data sets）的相关的实现。用户定义一个map函数来处理一个key/value对以生成一批中间的key/value对，再定义一个reduce函数将所有这些中间的有着相同key的values合并起来。很多现实世界中的任务都可用这个模型来表达。

 

Hadoop的Map/Reduce框架也是基于这个原理实现的，下面简要介绍一下Map/Reduce框架主要组成及相互的关系。

## 2.1       总体结构

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.1.1            Mapper和Reducer

运行于Hadoop的MapReduce应用程序最基本的组成部分包括一个*Mapper*和一个*Reducer*类，以及一个创建*JobConf*的执行程序，在一些应用中还可以包括一个*Combiner*类，它实际也是*Reducer*的实现。

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.1.2            JobTracker和TaskTracker

它们都是由一个master服务*JobTracker*和多个运行于多个节点的slaver服务*TaskTracker*两个类提供的服务调度的。master负责调度job的每一个子任务task运行于slave上，并监控它们，如果发现有失败的task就重新运行它，slave则负责直接执行每一个task。*TaskTracker*都需要运行在HDFS的*DataNode*上，而*JobTracker*则不需要，一般情况应该把*JobTracker*部署在单独的机器上。

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.1.3            JobClient

每一个job都会在用户端通过*JobClient*类将应用程序以及配置参数*Configuration*打包成jar文件存储在HDFS，并把路径提交到*JobTracker*的master服务，然后由master创建每一个*Task*（即*MapTask*和*ReduceTask*）将它们分发到各个*TaskTracker*服务中去执行。

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.1.4            JobInProgress

*JobClient*提交job后，*JobTracker*会创建一个*JobInProgress*来跟踪和调度这个job，并把它添加到job队列里。*JobInProgress*会根据提交的job jar中定义的输入数据集（已分解成*FileSplit*）创建对应的一批*TaskInProgress*用于监控和调度*MapTask*，同时在创建指定数目的*TaskInProgress*用于监控和调度*ReduceTask*，缺省为1个*ReduceTask*。

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.1.5            TaskInProgress

*JobTracker*启动任务时通过每一个*TaskInProgress*来launchTask，这时会把*Task*对象（即*MapTask*和*ReduceTask*）序列化写入相应的*TaskTracker*服务中，*TaskTracker*收到后会创建对应的*TaskInProgress*（此*TaskInProgress*实现非*JobTracker*中使用的*TaskInProgress*，作用类似）用于监控和调度该*Task*。启动具体的*Task*进程是通过*TaskInProgress*管理的*TaskRunner*对象来运行的。*TaskRunner*会自动装载job jar，并设置好环境变量后启动一个独立的java child进程来执行*Task*，即*MapTask*或者*ReduceTask*，但它们不一定运行在同一个*TaskTracker*中。

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.1.6            MapTask和ReduceTask

一个完整的job会自动依次执行*Mapper*、*Combiner*（在*JobConf*指定了*Combiner*时执行）和*Reducer*，其中*Mapper*和*Combiner*是由*MapTask*调用执行，*Reducer*则由*ReduceTask*调用，*Combiner*实际也是*Reducer*接口类的实现。*Mapper*会根据job jar中定义的输入数据集按<key1,value1>对读入，处理完成生成临时的<key2,value2>对，如果定义了*Combiner*，*MapTask*会在*Mapper*完成调用该*Combiner*将相同key的值做合并处理，以减少输出结果集。*MapTask*的任务全完成即交给*ReduceTask*进程调用*Reducer*处理，生成最终结果<key3,value3>对。这个过程在下一部分再详细介绍。

 

下图描述了Map/Reduce框架中主要组成和它们之间的关系：

![img](http://www.cppblog.com/images/cppblog_com/javenstudio/4165/o_hadoop-mapred.jpg)

 

## 2.2       Job创建过程

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.2.1            JobClient.runJob() 开始运行job并分解输入数据集

一个MapReduce的Job会通过*JobClient*类根据用户在*JobConf*类中定义的*InputFormat*实现类来将输入的数据集分解成一批小的数据集，每一个小数据集会对应创建一个*MapTask*来处理。*JobClient*会使用缺省的*FileInputFormat*类调用*FileInputFormat*.getSplits()方法生成小数据集，如果判断数据文件是isSplitable()的话，会将大的文件分解成小的*FileSplit*，当然只是记录文件在HDFS里的路径及偏移量和Split大小。这些信息会统一打包到jobFile的jar中并存储在HDFS中，再将jobFile路径提交给*JobTracker*去调度和执行。

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.2.2            JobClient.submitJob() 提交job到JobTracker

jobFile的提交过程是通过RPC模块（有单独一章来详细介绍）来实现的。大致过程是，*JobClient*类中通过RPC实现的Proxy接口调用*JobTracker*的submitJob()方法，而*JobTracker*必须实现JobSubmissionProtocol接口。JobTracker则根据获得的jobFile路径创建与job有关的一系列对象（即JobInProgress和TaskInProgress等）来调度并执行job。

 

*JobTracker*创建job成功后会给*JobClient*传回一个*JobStatus*对象用于记录job的状态信息，如执行时间、Map和Reduce任务完成的比例等。*JobClient*会根据这个*JobStatus*对象创建一个*NetworkedJob*的*RunningJob*对象，用于定时从*JobTracker*获得执行过程的统计数据来监控并打印到用户的控制台。

 

与创建Job过程相关的类和方法如下图所示

 

![img](http://www.cppblog.com/images/cppblog_com/javenstudio/4165/o_hadoop-mapred-class-2-2.jpg)

 

## 2.3       Job执行过程

上面已经提到，job是统一由*JobTracker*来调度的，具体的*Task*分发给各个*TaskTracker*节点来执行。下面通过源码来详细解析执行过程，首先先从*JobTracker*收到*JobClient*的提交请求开始。

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.3.1            JobTracker初始化Job和Task队列过程

#### 2.3.1.1     JobTracker.submitJob() 收到请求

当*JobTracker*接收到新的job请求（即submitJob()函数被调用）后，会创建一个*JobInProgress*对象并通过它来管理和调度任务。*JobInProgress*在创建的时候会初始化一系列与任务有关的参数，如job jar的位置（会把它从HDFS复制本地的文件系统中的临时目录里），Map和Reduce的数据，job的优先级别，以及记录统计报告的对象等。

#### 2.3.1.2     JobTracker.resortPriority() 加入队列并按优先级排序

*JobInProgress*创建后，首先将它加入到jobs队列里，分别用一个map成员变量*jobs*用来管理所有jobs对象，一个list成员变量jobsByPriority用来维护jobs的执行优先级别。之后JobTracker会调用resortPriority()函数，将jobs先按优先级别排序，再按提交时间排序，这样保证最高优先并且先提交的job会先执行。

#### 2.3.1.3     JobTracker.JobInitThread 通知初始化线程

然后*JobTracker*会把此job加入到一个管理需要初始化的队列里，即一个list成员变量jobInitQueue里。通过此成员变量调用notifyAll()函数，会唤起一个用于初始化job的线程*JobInitThread*来处理（JobTracker会有几个内部的线程来维护jobs队列，它们的实现都在*JobTracker*代码里，稍候再详细介绍）。*JobInitThread*收到信号后即取出最靠前的job，即优先级别最高的job，调用*JobInProgress*的initTasks()函数执行真正的初始化工作。

#### 2.3.1.4     JobInProgress.initTasks() 初始化TaskInProgress

Task的初始化过程稍复杂些，首先步骤*JobInProgress*会创建Map的监控对象。在initTasks()函数里通过调用*JobClient*的readSplitFile()获得已分解的输入数据的*RawSplit*列表，然后根据这个列表创建对应数目的Map执行管理对象*TaskInProgress*。在这个过程中，还会记录该*RawSplit*块对应的所有在HDFS里的blocks所在的DataNode节点的host，这个会在*RawSplit*创建时通过*FileSplit*的getLocations()函数获取，该函数会调用*DistributedFileSystem*的getFileCacheHints()获得（这个细节会在HDFS模块中讲解）。当然如果是存储在本地文件系统中，即使用*LocalFileSystem*时当然只有一个location即“localhost”了。

 

其次*JobInProgress*会创建Reduce的监控对象，这个比较简单，根据*JobConf*里指定的Reduce数目创建，缺省只创建1个Reduce任务。监控和调度Reduce任务的也是*TaskInProgress*类，不过构造方法有所不同，*TaskInProgress*会根据不同参数分别创建具体的*MapTask*或者*ReduceTask*。

 

*JobInProgress*创建完*TaskInProgress*后，最后构造*JobStatus*并记录job正在执行中，然后再调用*JobHistory*.*JobInfo*.logStarted()记录job的执行日志。到这里*JobTracker*里初始化job的过程全部结束，执行则是通过另一异步的方式处理的，下面接着介绍它。

 

与初始化Job过程相关的类和方法如下图所示

![img](http://www.cppblog.com/images/cppblog_com/javenstudio/4165/o_hadoop-mapred-class-2-3-1.jpg)

 

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.3.2            TaskTracker执行Task的过程

Task的执行实际是由*TaskTracker*发起的，*TaskTracker*会定期（缺省为10秒钟，参见MRConstants类中定义的HEARTBEAT_INTERVAL变量）与*JobTracker*进行一次通信，报告自己Task的执行状态，接收*JobTracker*的指令等。如果发现有自己需要执行的新任务也会在这时启动，即是在*TaskTracker*调用*JobTracker*的heartbeat()方法时进行，此调用底层是通过IPC层调用Proxy接口（在IPC章节详细介绍）实现。这个过程实际比较复杂，下面一一简单介绍下每个步骤。

#### 2.3.2.1     TaskTracker.run() 连接JobTracker

*TaskTracker*的启动过程会初始化一系列参数和服务（另有单独的一节介绍），然后尝试连接*JobTracker*服务（即必须实现*InterTrackerProtocol*接口），如果连接断开，则会循环尝试连接*JobTracker*，并重新初始化所有成员和参数，此过程参见run()方法。

#### 2.3.2.2     TaskTracker.offerService() 主循环

如果连接*JobTracker*服务成功，*TaskTracker*就会调用offerService()函数进入主执行循环中。这个循环会每隔10秒与*JobTracker*通讯一次，调用transmitHeartBeat()获得*HeartbeatResponse*信息。然后调用*HeartbeatResponse*的getActions()函数获得*JobTracker*传过来的所有指令即一个*TaskTrackerAction*数组。再遍历这个数组，如果是一个新任务指令即*LaunchTaskAction*则调用startNewTask()函数执行新任务，否则加入到tasksToCleanup队列，交给一个taskCleanupThread线程来处理，如执行*KillJobAction*或者*KillTaskAction*等。

#### 2.3.2.3     TaskTracker.transmitHeartBeat() 获取JobTracker指令

在transmitHeartBeat()函数处理中，*TaskTracker*会创建一个新的*TaskTrackerStatus*对象记录目前任务的执行状况，然后通过IPC接口调用*JobTracker*的heartbeat()方法发送过去，并接受新的指令，即返回值*TaskTrackerAction*数组。在这个调用之前，*TaskTracker*会先检查目前执行的Task数目以及本地磁盘的空间使用情况等，如果可以接收新的Task则设置heartbeat()的askForNewTask参数为true。操作成功后再更新相关的统计信息等。

#### 2.3.2.4     TaskTracker.startNewTask() 启动新任务

此函数的主要任务就是创建*TaskTracker$TaskInProgress*对象来调度和监控任务，并把它加入到runningTasks队列中。完成后则调用localizeJob()真正初始化Task并开始执行。

#### 2.3.2.5     TaskTracker.localizeJob() 初始化job目录等

此函数主要任务是初始化工作目录workDir，再将job jar包从HDFS复制到本地文件系统中，调用RunJar.unJar()将包解压到工作目录。然后创建一个RunningJob并调用addTaskToJob()函数将它添加到runningJobs监控队列中。完成后即调用launchTaskForJob()开始执行Task。

#### 2.3.2.6     TaskTracker.launchTaskForJob() 执行任务

启动Task的工作实际是调用TaskTracker$TaskInProgress的launchTask()函数来执行的。

#### 2.3.2.7     TaskTracker$TaskInProgress.launchTask() 执行任务

执行任务前先调用localizeTask()更新一下jobConf文件并写入到本地目录中。然后通过调用Task的createRunner()方法创建TaskRunner对象并调用其start()方法最后启动Task独立的java执行子进程。

#### 2.3.2.8     Task.createRunner() 创建启动Runner对象

*Task*有两个实现版本，即*MapTask*和*ReduceTask*，它们分别用于创建Map和Reduce任务。*MapTask*会创建*MapTaskRunner*来启动Task子进程，而*ReduceTask*则创建*ReduceTaskRunner*来启动。

#### 2.3.2.9     TaskRunner.start() 启动子进程真正执行Task

这里是真正启动子进程并执行Task的地方。它会调用run()函数来处理。执行的过程比较复杂，主要的工作就是初始化启动java子进程的一系列环境变量，包括设定工作目录workDir，设置CLASSPATH环境变量等（需要将*TaskTracker*的环境变量以及job jar的路径合并起来）。然后装载job jar包，调用runChild()方法启动子进程，即通过*ProcessBuilder*来创建，同时子进程的stdout/stdin/syslog的输出定向到该Task指定的输出日志目录中，具体的输出通过*TaskLog*类来实现。这里有个小问题，Task子进程只能输出INFO级别日志，而且该级别是在run()函数中直接指定，不过改进也不复杂。

 

与Job执行过程相关的类和方法如下图所示

![img](http://www.cppblog.com/images/cppblog_com/javenstudio/4165/o_hadoop-mapred-class-2-3-2.jpg)




## 2.4       JobTracker和TaskTracker

如上面所述，*JobTracker*和*TaskTracker*是MapReduce框架最基本的两个服务，其他所有处理均由它们调度执行，下面简单介绍它们内部提供的服务及创建的线程，详细过程下回分解J

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.4.1            JobTracker的服务和线程

*JobTracker*是MapReduce框架中最主要的类之一，所有job的执行都由它来调度，而且Hadoop系统中只配置一个*JobTracker*应用。启动*JobTracker*后它会初始化若干个服务以及若干个内部线程用来维护job的执行过程和结果。下面简单介绍一下它们。

 

首先，*JobTracker*会启动一个interTrackerServer，端口配置在*Configuration*中的"mapred.job.tracker"参数，缺省是绑定8012端口。它有两个用途，一是用于接收和处理*TaskTracker*的heartbeat等请求，即必须实现*InterTrackerProtocol*接口及协议。二是用于接收和处理*JobClient*的请求，如submitJob，killJob等，即必须实现*JobSubmissionProtocol*接口及协议。

 

其次，它会启动一个infoServer，运行*StatusHttpServer*，缺省监听50030端口。是一个web服务，用于给用户提供web界面查询job执行状况的服务。

 

*JobTracker*还会启动多个线程，*ExpireLaunchingTasks*线程用于停止那些未在超时时间内报告进度的Tasks。*ExpireTrackers*线程用于停止那些可能已经当掉的*TaskTracker*，即长时间未报告的*TaskTracker*将不会再分配新的Task。*RetireJobs*线程用于清除那些已经完成很长时间还存在队列里的jobs。*JobInitThread*线程用于初始化job，这在前面章节已经介绍。*TaskCommitQueue*线程用于调度Task的那些所有与*FileSystem*操作相关的处理，并记录Task的状态等信息。

 

[回到顶部](https://www.cnblogs.com/jycjy/p/6741808.html#_labelTop)

### 2.4.2            TaskTracker的服务和线程

*TaskTracker*也是MapReduce框架中最主要的类之一，它运行于每一台*DataNode*节点上，用于调度Task的实际运行工作。它内部也会启动一些服务和线程。

 

TaskTracker也会启动一个StatusHttpServer服务来提供web界面的查询Task执行状态的工具。

其次，它还会启动一个taskReportServer服务，这个用于提供给它的子进程即TaskRunner启动的MapTask或者ReduceTask向它报告状况，子进程的启动命令实现在TaskTracker$Child类中，由TaskRunner.run()通过命令行参数传入该服务地址和端口，即调用TaskTracker的getTaskTrackerReportAddress()，这个地址会在taskReportServer服务创建时获得。

TaskTracker也会启动一个MapEventsFetcherThread线程用于获取Map任务的输出数据信息。

[转载](http://www.cppblog.com/javenstudio/articles/43073.html)
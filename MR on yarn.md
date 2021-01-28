# MR on YARn

[TOC]

## 架构图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190802110347904.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA0NTIzODg=,size_16,color_FFFFFF,t_70)
RM: ResourceManager
NM: NodeManager

1.用户向yarn提交job，其中包含Application master程序，以及启动Application master的脚本等

2.RM为该job分配第一个Container，与对应的NM通信，要求他在这个Container启动作业的Application master

3.Application master向Application manager注册信息，这样用户就可以通过RM的web界面查看job的状态

4.application master 采用轮询的机制通过【RPC】协议向Resource Scheduler申请和认领资源

5.一旦申请到资源，与对应的NM通信，请求启动task

6.NM设置好运行环境之后，将任务的启动脚本写到一个脚本中，并通过该脚本启动任务，开始运行任务

7.每个task通过【RPC】协议向application Master回报自己的状态和进度，以让application master掌握各个任务的状态，从而在任务失败的时候，重新启动

8.job运行完成后，application master向application manager注销并关闭自己
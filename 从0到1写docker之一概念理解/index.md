# 从0到1写docker之一概念理解


<!--more-->

# (一)概念理解

[项目地址](https://github.com/lance-e/gocker)
欢迎folk，star，follow🥰，有问题欢迎提issue，pr


## docker 三大核心技术：

##### Namespace:



- UTS Namespace 主要用来隔离 nodename 和 domainname 两个系统标识

- IPC Namespace 用来隔离 System V IPC 和 POSIX message queues(进程间通信)

- PID Namespace是用来隔离进程 ID的

- MountNamespace用来隔离各个进程看到的挂载点视图

- User Namespace 主要是隔离用户的用户组 ID

- Network Namespace 是用来隔离网络设备、 IP地址端口 等网络械的 Namespace。



##### Cgroups:

四个重要的概念：tasks，cgroup，hierarchy，subsystem

- tasks:就是一个进程
- cgroup:控制族群，也就是一个按某种规定划分的进程组，cgroups使用的资源都是以控制族群为单位划分的
- hierarchy:层级，是由多个cgroup组成的树状关系，
- subsystem:资源子系统，相当于资源控制器，如cpu，memory子系统，必须附加到某个层级hierarchy才能起作用，



##### union file system:

把其他文件系统联合到一个联合挂载点的文件系统服务，具有写时复制和联合挂载的特性

- 现在docker多采用的是overlayFS，包含四个部分，upper，lower，work，merged

## 参考文档：

《自己动手写docker》

https://blog.csdn.net/qq_31960623/article/details/120260769

https://www.cnblogs.com/crazymakercircle/p/15400946.html#autoid-h3-2-2-0



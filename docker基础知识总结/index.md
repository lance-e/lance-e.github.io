# Docker基础知识总结


<!--more-->

# docker 基础知识

### 什么是docker？

Docker!=轻量级虚拟机
Docker使用的是容器化技术，docker利用的是宿主机的操作系统内核
Docker是一个Client-Server结构的系统，Docker守护进程运行在主机上， 然后通过Socket连接从客户端访问Docker守护进程。
Docker守护进程从客户端接受命令，并按照命令，管理运行在主机上的容器。
一个docker 容器，是一个运行时环境，可以简单理解为进程运行的集装箱。

### docker 命令简介

熟能生巧，这里仅做总结大部分命令作用，其实见名知意

~~~bash
//常见命令
//容器操作
//容器操作命令都可以加container,例：docker container run XXX
docker attach       //将本地标准输入、输出和错误流附加到正在运行的容器，通俗的就是可以进入容器
docker run          //创建并运行一个容器,选项-d：可以让容器在后台运行
docker exec         //进入一个容器,选项-it：保持标准输入打开，并分配一个伪终端
docker stats        //展示运行中的容器的状态
docker start        //启动一个或多个停止的容器
docker stop         //停止一个或多个运行的容器
docker kill         //杀死一个或多个运行的容器
docker ps           //展示正在运行的容器
docker rm           //删除容器
docker rmi          //删除镜像
docker rename       //重命名容器
docker restart      //重启一个或多个容器
docker logs         //记录一个容器的日志
docker top          //展示运行中容器的进程信息
docker export       //导出容器，通常与import联用
docker import       //导入容器，通常与export联用
docker pause        //暂停一个或多个容器的所有进程
docker unpause      //解除一个或多个容器的所有暂停的进程
docker port         //展示容器的端口映射关系
docker update       //更新容器的配置文件
docker wait         //阻塞直到一个或多个容器停止，打印它们的退出代码
docker commit       //通过一个容器生成一个镜像
docker cp           //从容器和本地主机之间复制文件或文件夹
docker diff         //展示容器内文件系统的变更




//镜像操作
docker build        //通过dockerfile生成一个镜像
docker images       //展示所有镜像
docker pull         //从dockerHub中拉取镜像
docker push         //将镜像推送到dockerHub
docker search       //搜索镜像
docker save         //存入：将一个或多个镜像打包为tar格式
docker load         //加载：加载镜像保存文件，使生成镜像
docker tag          //创建指定镜像的标签
(携带image子命令)
docker image build  //通过dockerfile生成一个镜像
docker image ls     //展示所有镜像
docker image pull   //拉取镜像
docker image push   //推送镜像
docker image history//展示一个镜像的历史
docker image import //从tarball导入内容以创建文件系统映像
docker image inspect//展示一个或多个镜像的的详细信息
docker image load   //加载：加载镜像保存文件，使生成镜像
docker image save   //存入：将一个或多个镜像打包为tar格式
docker image prune  //删除不用的镜像
docker image rm     //删除一个或多个镜像
docker image tag    //创建指定镜像的标签
 

//...
~~~

ok啊～感觉就这些命令常用，没列出来的，还有一些详细参数就查官网吧，官网更详细。

### docker 的核心技术

先看看AI怎么说？

~~~Docker的核心技术包括以下几个方面：
Linux容器（Linux Containers，LXC）：Docker利用Linux内核提供的容器化技术，实现了隔离进程、文件系统、网络和资源的轻量级虚拟化。Linux容器是Docker实现容器化的基础。

命名空间（Namespaces）：Docker使用命名空间来隔离进程的运行环境，包括PID（进程ID）、网络、IPC（进程间通信）、挂载点和UTS（主机名和域名）等。每个容器都有自己独立的命名空间，使得容器内的进程感知不到宿主机或其他容器的存在。

控制组（Control Groups，cgroups）：Docker利用控制组来限制和分配容器可使用的资源，如CPU、内存、磁盘和网络带宽等。控制组可以对容器进行资源配额和优先级管理，确保容器之间互不干扰。

联合文件系统（UnionFS）：Docker使用联合文件系统将容器的文件系统与宿主机的文件系统进行层叠并共享，以实现镜像的高效存储和容器的快速启动。常用的联合文件系统有AUFS、OverlayFS和DeviceMapper等。

Docker镜像（Docker Image）：Docker镜像是容器运行时的可执行文件，包含了运行一个容器所需的文件系统、依赖库和配置等。镜像可以通过Dockerfile定义和构建，也可以从Docker Hub等镜像仓库中获取。

Docker容器（Docker Container）：Docker容器是基于镜像创建的运行实例，可以独立运行在宿主机上，并与其他容器隔离。容器可以方便地启动、停止、暂停和销毁，提供了一种轻量级、可移植的应用部署方式。

这些核心技术共同构成了Docker的基础架构，使得Docker能够实现高效、可移植、可扩展的容器化应用部署和管理。
~~~

(总结到位，有些词汇我也不懂😭)
那么可以看出docker的三大底座：**Namespaces,control Groups(cgroups),UnionFS**

##### 一.Namespaces(命名空间)

Namespase 是linux内核提供的隔离进程资源的方案，虚拟资源的隔离
主要分为了7类
1.PID命名空间：隔离了进程id
2.Mount命名空间：隔离文件系统挂载点
3.Network命名空间：隔离网络设备，ip地址，路由表，防火墙等
4.UTS命名空间：隔离主机和域名
5.IPC命名空间：隔离 System V IPC 和 POSIX 消息队列(这个是啥？没学有点懵)，(IPC是linux进程间数据交互和同步的方式)
6.User命名空间：隔离用户和用户组
7.Cgroup命名空间：隔离进程组间的资源

###### PID隔离(没看懂，以后补充)

###### 网络隔离：

docker 提供的四种网络模式：host,container,none,bridge
默认为网桥模式
在该模式下，除了会分配隔离的网络命名空间，docker还会给容器分配ip地址
docker服务器在主机启动后，会创建一个新的docker0虚拟网桥，之后所有的服务默认情况下都会连接到该虚拟网卡
默认情况下，每个容器在创建的时候都会创建一对虚拟网卡，这一对构成一个数据通道，其中一个网卡会放在容器中，加入到docker0网桥中.
docker0会为每个容器分配一个不同的ip地址，并将docker0点ip地址设置为默认网关
网桥 docker0 通过 iptables 中的配置与宿主机器上的网卡相连，所有符合条件的请求都会通过 iptables 转发到 docker0 并由网桥分发给对应的机器。
libnetwork:有待补充
容器网络模型：有待补充

##### 二.CGroups(控制组)

CGroups用户隔离宿主机的物理资源，例如 CPU、内存、磁盘 I/O 和网络带宽。
每一个CGroup都是一组拥有相同属性的进程，CCroup是有层级关系的，子类可以继承父类的隔离标准
(也是挺复杂的，以后补充)

##### 三.UnionFS(联合文件系统)

什么是镜像？
image是docker部署的基本单位，包含程序文件，以及运行所需的资源文件，对外表现为文件的形式(mount点形式)
什么是UnionFS？
UnionFS是linux点文件系统技术，用于将多个文件系统挂载到同一个挂载点
AUFS：Advanced UnionFS，UnionFS先进版,提供更优秀的性能和效率

##### 四.总结：

docker = LXC + AUFS(docker = linux container + advance UnionFS)
LXC:用来进行资源管理
AUFS:用来进行镜像管理

###### cgroup，LXC，docker三者的关系：

cgroup ----->LXC------>docker   进行层层封装

### DockerFile的编写

DockerFile 是文本文件， 其中的每条指令都会建立一层，每条指令就对应这一层该如何构建

##### FROM指定基础镜像

FROM是必备指令，也是第一条指令
可以选择现有的镜像，也可以使用scratch镜像，代表空镜像，这就意味着你不选择任何镜像作为你的基础镜像，那么接下来所写的指令将作为镜像的第一层存在
例如go开发的应用，编译之后直接生成的就是可执行的2进制文件，不需要依赖操作系统，所依赖的库都包含在了可执行文件中，所以很适合使用FROM scratch，让镜像体积变得小巧

##### RUN执行命令行命令

RUN指令是用来执行命令行命令的

~~~txt
shell格式：
RUN <命令>

exec格式：
RUN ["可执行文件","参数A","参数B",...]
~~~

因为每条指令对应一层，所以尽量把多条shell命令写在同一条RUN指令中，使用&&连接，可以防止产生臃肿，多层的镜像
Dockerfile 支持 Shell 类的行尾添加 \ 的命令换行方式，以及行首 # 进行注释的格式。
在每一层的构建结尾，记得删除无关文件，也可以避免臃肿的镜像产生

##### 构建镜像

~~~bash
docker build [OPTIONS] PATH | URL | -
~~~

例如：docker build -t nginx:v3 .

##### 镜像构建上下文(context)

通过上面的例子可以看到最后有一个 . ，. 表示当前目录，**注意不要误认为是DockerFile目录**，这是在指定上下文路径。

###### 什么是上下文呢？

docker 在运行时分为docker引擎(服务端守护进程)和客户端工具,docker引擎提供了一组REAT API(docker remote API)，客户端工具通过api远程调用服务端的服务，让我们以为各种docker服务是在本机进行的，实际上是在远程的服务端进行的，这就让我们操控远程服务端引擎更加便捷。
但是，我们构建DockerFile时可不止使用RUN指令，还需要一些本地文件，但是在这样c/s架构下，该如何将本地文件提供给远程服务器？
所以就引入了上下文这一概念，通过构建上下文，当执行docker build 命令时，就会根据构建的上下文，将所需的本地文件打包，上传到docker引擎，docker引擎展开上下文包就可以获得所有构建所需的文件

~~~txt
COPY ./package.json /app/
~~~

这里的意思就是把**上下文(context)**目录下的package.json文件复制,而不是DockerFile或其他目录下
一般将DockerFile放在空目录下或项目目录下，如何该目录下没有所需文件，需要把文件复制一份，如果有不想上传给docker引擎的文件，可以通过.dockerignore指定不想上传的文件，把该文件从上下文中剔除(类似于.gitignore)
DockerFile的命名不一定一定是DockerFile，也不一定是在上下文目录下

##### 其他的 docker build 用法：

###### 1.使用git repo构建

~~~bash
docker build [OPTIONS] PATH | URL | -
~~~

我们可以看到docker build 不止可以从上下文目录中构建，还可以从URL中构建，也就是说还可以从git仓库中构建，那么我们将代码上传到 GitHub上，就可以使用docker快速部署

###### 2. 使用tar压缩包

如果URL中是一个tar压缩包，docker会去下载压缩包，并解压后构建镜像

###### 3.从标准输入读取DockerFile进行构建(以后补充)

###### 4.从标准输入读取上下文压缩包进行构建(以后补充)

### DockerFile指令详解

除了以上提到的FROM，RUN基础指令外，还有很多需要的指令：

##### COPY

复制文件
格式：

~~~txt
COPY [--chown=<user>:<group>] <源路径>... <目标路径>
COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]
~~~

源路径可以是多个，支持通配符，目标路径可以是工作目录的相对路径，也可以是绝对路径，使用相对路径需要使用WORKDIR指令指定工作目录。--chown=<user>:<group> 选项来改变文件的所属用户及所属组。

##### ADD

也是复制文件，但是比COPY高级
格式：与COPY一样
当复制一个压缩包时，使用ADD可以进行自动解压的功能(如果本来就想把压缩包复制到目标目录，而不解压缩，那就不要用ADD指令)

##### CMD

容器启动命令
格式：

~~~txt
shell格式
CMD <命令>
exec格式
CMD ["可执行文件","参数A","参数B"...]
参数列表格式
CMD ["参数A","参数B"...] (指定了ENTRYPOINT后，CMD就用来指定参数)
~~~

在**运行时也可以指定新的命令来替换镜像设置的默认命令**
使用shell格式的命令，实际的命令会被包装为 sh -c 的参数的形式进行执行。
一般格式推荐使用exec格式，指令会被解析为json数组，所以一定要使用**双引号**
容器前台运行和后台运行的问题：
因为容器内没有后台服务的概念，容器是为了主进程存在的，主进程退出，那么容器也会退出，那么执行shell格式命令时，主进程实际上是sh ，当该条命令结束后，sh也就结束了，主进程退出，那么容器就退出了，所以我们应该使用exec格式命令直接执行可执行文件，并在前台运行

##### ENTRYPOINT 入口点

ENTRYPOINT格式与RUN格式一样，分为shell和exec格式
ENTRYPOINT也是用来指定容器启动程序和启动参数，可以在运行时替代，需要通过 docker run 的参数 --entrypoint 来指定
当指定了ENTRYPOINT后，CMD指令就是用来指定参数，实际执行时变成了：

~~~txt
ENTRYPOINT <CMD>
~~~

###### 应用场景一：让镜像想命令一样使用

###### 应用场景二：应用运行前的准备工作

##### ENV 设置环境变量

就是用来设置环境变量的
格式：

~~~txt
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...
~~~

##### ARG 构建参数

也是用来设置环境变量的，但是与ENV有区别：ARG构建的环境变量在容器运行时是不存在的
格式：

~~~txt
ARG <参数名>[=<默认值>]
~~~

Dockerfile 中的 ARG 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 docker build 中用 --build-arg <参数名>=<值> 来覆盖。
ARG指令是有生效范围的，如果在FROM前使用，那么定义的参数只能在FROM指令中使用，如果要在FROM后也使用，那么就要在FROM后也要再使用ARG指令，分别指定变量

##### VOLUME 定义匿名卷

VOLUME指令可以指定某些目录为匿名卷，将动态数据存储在匿名卷中，可以避免用户忘记指定数据卷，使容器正常运行，而不会将动态数据过多的存储在容器存储层
格式：

~~~txt
VOLUME ["路径1","路径2"]
VOLUME <路径>
~~~

##### EXPOSE 暴露端口

格式：

~~~txt
EXPOSE <端口一> [<端口二>...]
~~~

EXPOSE指令用于声明端口，也只是用于声明容器打算使用什么端口而已，镜像启动时也不会因为这个声明就启动该端口服务，只是用于方便镜像使用者对容器进行端口映射，另外还可以在运行时随机端口映射时，就是使用docker run -P时，可以随机端口映射到EXPOSE指定的端口

##### WORKDIR 指定工作目录

WORKDIR用于指定工作目录(或者成为当前目录),就是pwd的命令展示的工作目录
格式：

~~~txt
WORKDIR <工作目录>
~~~

如果WORKDIR使用的是相对路径，那么所切换的路径与之前WORKDIR有关

##### USER 指定用户

使用USER指令时，确保用户存在
格式：

~~~txt
USER <用户名>[:<用户组>]
~~~

##### HEALTHYCHECK 健康检查

格式：

~~~txt
HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令
HEALTHCHECK NONE：如果基础镜像有健康检查指令，使用这行可以屏蔽掉其健康检查指令
~~~

##### ONBUILD 当前镜像作为基础镜像执行

ONBUILD指令所指定的指令在当前镜像构建时不会执行，之后当当前镜像作为基础镜像时，才会执行
格式：

~~~txt
ONBUILD <其他命令>
~~~

##### LABEL 添加元数据

LABEL以减值对的形式给镜像添加元数据

~~~txt
LABEL <key>=<value> <key>=<value> <key>=<value> ...
~~~

##### SHELL 指令

用于指定RUN CMD ENTRYPOINT指令的shell,Linux 中默认为 ["/bin/sh", "-c"]
格式：

~~~txt
SHELL ["executable", "parameters"]
~~~

参考资料：
[蓝山第10节课课件](https://github.com/LanshanTeam/Courseware-Backend-Go-2023/tree/main/lesson10)
[docker技术入门与实战](https://www.cnblogs.com/crazymakercircle/p/15400946.html#autoid-h3-7-3-0)
[docker入门到实践](https://yeasy.gitbook.io/docker_practice/image/build)


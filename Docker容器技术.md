# Docker容器技术

### Docker介绍

Docker是一个虚拟化技术，能够快速的部署应用程序，免去了配置环境的复杂。

应用程序运行在容器(Container)中，独立于其他容器与外部环境，但是容器中的应用运行是寄托于宿主操作系统的，实际上依然是在直接使用操作系统的资源，这也是不同于虚拟机的地方，开销也会小很多。

Docker的方便之处在于它开箱即用的思想。应用程序的环境配置、基本运行指令都封装在了镜像（Image）中，我们需要使用时，只需要根据对应的镜像创建容器，即可随时随地运行应用程序。

### 基本概念

#### 容器

简单地说，容器是机器上的一个沙盒进程，它与主机上的所有其他进程隔离。这种隔离利用了Linux中的内核名称空间和cgroup功能。

- 是映像的可运行实例。可以使用DockerAPI或CLI创建、启动、停止、移动或删除容器。
-  可以在本地机器、虚拟机上运行，也可以部署到云中。 
- 是可移植的（可以在任何操作系统上运行）。 
- 与其他容器隔离，并运行自己的软件、二进制文件和配置。

#### 镜像

简单来说，镜像提供了容器运行时所需要的环境以及命令。

当运行容器时，它使用一个独立的文件系统。这个自定义文件系统是由容器镜像提供的。它还必须包含运行应用程序所需的一切：所有依赖项、配置、脚本、二进制文件等。镜像还包含了容器的其他配置，如环境变量、要运行的默认命令和其他元数据。

镜像是一种分层的结构，构建镜像时的每一条指令都会转换成一个镜像层，并且，这些镜像层是可以重复使用的（缓存），考虑到镜像会被多次使用，镜像中的数据是只读的，数据的修改在容器层，各个层次之间的文件是一种覆盖关系，即：

- 文件读取：要读取一个文件，Docker会最上层往下依次寻找，找到后则打开文件。
- 文件创建和修改：创建新文件会直接添加到容器层中，修改文件会从上往下依次寻找各个镜像中的文件，如果找到，则将其复制到容器层，再进行修改。
- 删除文件：删除文件也会从上往下依次寻找各个镜像中的文件，一旦找到，并不会直接删除镜像中的文件，而是在容器层标记这个删除操作。

![image-20230610141751278](/Users/renchan/Library/Application Support/typora-user-images/image-20230610141751278.png)

#### 仓库（Repository）

存放镜像的位置

#### 注册服务器(Register)

存放仓库的服务器

### 创建镜像

有两种创建镜像的方式，第一种是使用`commit`命令来完成，是对编辑过后的容器构建成镜像。第二种是通过dockerFile进行编写构建。推荐使用第二种方式。

#### commit

`docker commit 容器名称/ID 新的镜像名称`

以在ubuntu镜像中安装java环境为例

`docker run -it --name u ubuntu:latest`

在输入界面中输入

`apt update`

`apt install openjdk-8-jdk`

`exit`

然后执行`docker commit u1 u-Java`

![image-20230610152135355](/Users/renchan/Library/Application Support/typora-user-images/image-20230610152135355.png)

#### Dockerfile

这里以本人手上的一个博客项目的镜像编写为例

```dockerfile
FROM u-java # 每一个镜像都要从from开始，代表使用的base镜像 如果没有 使用 FROM scratch代表空镜像
COPY target/aurora-springboot-0.0.1.jar app.jar # 复制jar包到容器内部
CMD java -jar app.jar # CMD指令代表在容器启动后执行的指令
```

`docker build -t 镜像名称 构建目录  ` `docker build -t app1.0 .  `

`docker run --name app-test -d -p 8081:8081 app1.0` 运行容器 其中-p为端口映射

![image-20230610161411084](/Users/renchan/Library/Application Support/typora-user-images/image-20230610161411084.png)

##### 常用命令说明

下面介绍一下Dockerfile里面的常用命令

- 指令COPY 复制   COPY 宿主机文件 容器内文件

- 指令ADD，它跟COPY命令类似，也可以复制文件到容器中，但是它可以自动对压缩文件进行解压，这里只需要将压缩好的文件填入即可
- 指令VOLUME，就像我们使用`-v`参数一样，会创建一个挂载点在容器中。

### 发布镜像

首先在Docker Hub的仓库中创建一个公有仓库

![image-20230610163546060](/Users/renchan/Library/Application Support/typora-user-images/image-20230610163546060.png)

然后在命令行中进行登录操作

`docker login -u "name"`

成功之后，为了命名的规范，我们把镜像的名字进行一下修改

`docker tag 镜像名 用户名/仓库名 `

最后进行推送

`docker push 镜像名`

![image-20230610163750070](/Users/renchan/Library/Application Support/typora-user-images/image-20230610163750070.png)

### 容器网络

Docker自带有3种网络类型 分别是环回网络、桥接网络以及共享网络 创建的容器默认使用桥接网络

`docker network ls` 可以查看当前docker的网络

在创建容器的时候，可以使用`--network=none`来指定网络类型

- None 环回网络 即本地网络，不能与外界通信
- bridge 桥接网络 是docker创建的虚拟网络，连入桥接网络的容器就会接入这个虚拟网络中，并且通过docker创建的docker0网络与外界连接，也就是连接着虚拟子网与宿主机。docker0可以看作是路由器的作用。
- host网络 共享网络，与宿主机使用同一个网络。只要宿主主机能连接到互联网，容器内部也是可以直接使用的。传输性能基本没有什么折损，而且我们可以直接开放端口等，不需要进行任何的桥接。

#### 自定义网络

docker支持用户自己创建一个网络，并且提供了3种网络驱动,注意不同的网络之间是隔离的

- bridge 就是桥接网络
- Overlay 覆盖网络，用于跨主机方案
- macvlan 用于跨主机方案

这里就介绍下bridge网络的创建

`docker network create --driver bridge test` 创建网络,这个网络与自带的网络处于不同的网络，所以两个网络之间不能进行直接通信

`docker run -it --network=test --name=u2 ubuntu-net` 指定容器的网络

` docker network connect test u2` 将容器连接到某个新的网络，这样容器就可以和新的网络之间的设备进行通信了

`--ip` 指定容器的ip地址

Docker提供了DNS服务器，它就像是一个真的DNS服务器一样，能够对域名进行解析，使用很简单，我们只需要在容器启动时给个名字就行了，我们可以直接访问这个名称，最后会被解析为对应容器的IP地址，但是注意只会在我们用户自定义的网络下生效，默认的网络是不行的。

我们也可以让两个容器同时共享同一个网络，注意这里的共享是直接共享同一个网络设备，两个容器共同使用一个IP地址，只需要在创建时指定。

`docker run -it --name=test01 --network=container:test02 ubuntu-net`

在桥接模式下，为了能够让外部能够访问到容器内部的服务，我们需要进行端口映射

`-p 80:79`代表将宿主机的80端口映射到容器的79端口，这样外部访问宿主机的80端口时，就是访问了容器的79端口

### 持久化存储

一般来说 我们创建的容器在运行时产生的所有数据，都是存放在容器内部文件中的，当我们删除容器时，数据也就随之删除了，但是我们可以指定路径映射，将某些我们需要的数据进行持久化存储。

#### 文件的挂载

在运行容器时，通过`-v`命名指定映射路径 格式为： `-v 宿主机路径:容器内路径`

这样我们对挂载目录中的文件进行编辑，那么相当于编辑的是宿主主机的数据，反之亦然。

当我们销毁容器时，数据并不会被清除掉，可以实现持久化。



注意如果我们在使用`-v`参数时不指定宿主主机上的目录进行挂载的话，那么就由Docker来自动创建一个目录，并且会将容器中对应路径下的内容拷贝到这个自动创建的目录中，最后挂在到容器中，这种就是由Docker管理的数据卷了（docker managed volume）

如下图所示，这里就是docker自动创建的路径映射目录

![image-20230615203358026](/Users/renchan/Library/Application Support/typora-user-images/image-20230615203358026.png)

#### 部分文件的复制

有时候可能并不需要直接挂载一个目录上去，仅仅是从宿主主机传递一些文件到容器中，这里我们可以使用`cp`命令来完成：

`docker cp 文件或者文件夹 容器id或者name:路径` 

#### 容器间的数据共享

- 第一种方式，是指定同一个目录进行挂载，也可以将另一个容器挂载的目录，直接在启动容器时指定使用此容器挂载的目录：

```
docker run -it -v ~/test:/root/test --name=data_test ubuntu-volume
docker run -it --volumes-from data_test ubuntu-volume //--volumes-from指定另一个容器
```

这里我们通过`--volumes-from `指定了另一个容器，这种用于给其他容器提供数据卷的容器，我们一般称为数据卷容器。数据卷容器中挂载的内容，在当前容器中也是存在的，并且就算此时数据卷容器被删除，那么也不会影响到这边，因为这边相当于是继承了数据卷容器提供的数据卷，所以本质上还是让两个容器挂载了同样的目录实现数据共享。

- 创建一个单纯存放数据的容器，直接将容器中打包好的数据分享给其他容器。

首先是要构建一个镜像

```
FROM ubuntu
COPY target/aurora-springboot-0.0.1.jar /root/abc/app.jar
VOLUME /root/abc
```

然后运行容器

`docker run --it --name=data_test v_test`

这样就创建了一个数据卷容器，他的映射路径由docker管理。

![image-20230615211357072](/Users/renchan/Library/Application Support/typora-user-images/image-20230615211357072.png)

当我们想要在别的容器使用这个数据的时候，只需要：

`docker run -p 80:80 --volumes-from=data_test -d nginx`

这样我们就实现了将数据放在容器中进行共享，我们不需要刻意去指定宿主主机的挂载点，而是Docker自行管理，这样就算迁移主机依然可以快速部署。



### 资源管理

#### 日志查看

`docker logs -f 容器名`

#### 容器监控

`docker stats` 对容器运行时占用的各种资源进行查看

`docker top 容器ID/名称` 内存占用最高的容器排序查看

#### 物理资源管理

限制分配给容器的机器资源

`docker run -m 内存限制 --memory-swap=内存和交换分区总共的内存限制 镜像名称`

`docker run -c 1024 ubuntu` 调整容器占用的cpu权重

`docker run -it --cpus=1 ubuntu` 限制占用的核心数

`docker run -it --cpuset-cpus=0,1 ubuntu`指定使用核心序号

`--device-read/write-bps=/dev/sha:10MB`和`--device-read/write-iops`限制磁盘io性能

### 常用命令总结

在终端内使用ctrl+p ctrl+q 可退出当前终端，而不是终止容器

`docker attach 容器ID/名称`回到容器终端

`docker logs -f 容器名` 查看容器输出日志 -f持续输出

`docker exec [options] 容器name COMMAND [args...]` 用于对正在运行的容器执行命令 最为常用的为在容器中新建一个终端：`docker exec -it tomcat bash`

 `docker pause/kill/unpause test`

`docker pull 镜像` 拉取镜像

`docker rmi` 删除镜像

`docker rm` 删除容器

`docker vloume ls`查看所有的数据卷容器

`docker ps -a`查看容器

`docker images`查看镜像

`docker inspect 镜像/容器/网络`查看元数据 

### Docker Compose

是一种容器编排技术 帮助我们管理一系列容器

通过docker compose 我们可以把一系列容器全部部署，比较方便



### 参考文献

https://juejin.cn/post/7021006271818137630?share_token=878fa440-84d9-4d8a-980e-aaba73440daf

https://docs.docker.com/get-started/

https://blog.csdn.net/qq_25928447/article/details/125643311?spm=1001.2014.3001.5501

https://haicoder.net/docker/docker-inspect.html

https://blog.csdn.net/qq_38567039/article/details/125775433
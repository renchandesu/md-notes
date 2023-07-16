# CENTOS安装redis

### 系统

腾讯云 CENTOS7.9

### 具体步骤

#### 安装gcc

如果没有，自行安装

`yum install -y gcc`

#### 下载安装包

下载的是redis7

`wget https://download.redis.io/releases/redis-7.0.2.tar.gz`

解压

`tar -zxf redis-7.0.2.tar.gz `

#### make

进入解压后的目录，执行

`make`

`make install`

默认安装在`/usr/local/bin`目录下

#### 配置环境变量

如果想在任何地方都启动redis-server 则需要把/usr/local/bin加入环境变量中

`vi /etc/profile`

在末尾添加：(root)

```
PATH=$PATH:/usr/local/bin/
export RPATH
```

然后

`source /etc/profile `

之后就ok 你可以在任何地方使用`redis-cli` `redis-server`这些命令

#### 配置conf文件

如果需要配置一些选项，则在源码目录下找到redis.conf文件，修改一些配置项，然后运行redis时，

`redis-server redis.conf` 就能让配置文件生效

一些基本配置：

```
port 6379
bind 0.0.0.0 让远程主机能够访问
rotected-mode no 保护模式关闭 然后可以使用密码登陆
requirepass password 设置密码 不设置密码，客户端无法连接

```


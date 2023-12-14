# 对象存储与MinIO
## 对象存储
### 介绍
### 与传统存储方式的比较
### 一些基本概念
#### 对象(object)
对象是 OSS 存储数据的基本单元，也被称为OSS的文件。
对象的组成：
- 元信息（Object Meta）
- 用户数据（Data）
- 文件名（Key）组成。
key是对象在存储空间内部的唯一标识
#### 存储空间(bucket)
用于存储对象（Object）的容器，所有的对象都必须隶属于某个存储空间。一个bucket类似于文件系统中的文件夹或者目录,一个桶可以拥有任意多个对象

## MinIO
>Minio是GlusterFS创始人之一Anand Babu Periasamy发布新的开源项目。基于Apache License v2.0开源协议的对象存储项目，采用Golang实现，客户端支Java,Python,Javacript, Golang语言。

MinIO设计的主要目标是作为私有云对象存储的标准方案。主要用于存储海量的图片，视频，文档等。非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几kb到最大5T不等。

开源，免费使用，提供linux macos windows操作系统的软件包，以及docker镜像，部署非常的方便

### 安装
因为minio提供了docker镜像，所以可以使用Docker直接进行部署，非常的方便
- linux/macos
```
mkdir -p ~/minio/data
docker run \
   -p 9000:9000 \
   -p 9090:9090 \
   --name minio \
   -v ~/minio/data:/data \
   -e "MINIO_ROOT_USER=ROOTNAME" \
   -e "MINIO_ROOT_PASSWORD=CHANGEME123" \
   quay.io/minio/minio server /data --console-address ":9090"
```
- windows
```
docker run \
   -p 9000:9000 \
   -p 9090:9090 \
   --name minio1 \
   -v D:\minio\data:/data \
   -e "MINIO_ROOT_USER=ROOTUSER" \
   -e "MINIO_ROOT_PASSWORD=CHANGEME123" \
   quay.io/minio/minio server /data --console-address ":9090"
```
-p 端口映射
-v 路径映射，minio的/data文件夹中的数据将存放在映射的目录中
-e 设置环境变量，只当root用户的的用户名与密码,注意用户名长度最少3 密码最少8 
--console-address ":9090" 指定客户端端口为9090
### 登录控制台
打开防火墙的9090端口，即可访问
![[Pasted image 20230830151141.png]]
### 创建bucket
![[Pasted image 20230830151236.png]]


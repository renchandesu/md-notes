# Linux服务器安装MYSQL数据库、MINIO部署、JAVA环境配置

最近在做一个spring boot的项目，因为想部署到服务器上，所以需要配置各种各样的环境，故做一个总结，也算是一个备忘录吧~

## Mysql数据库

**注：以下命令请在前加上sudo 或者root身份安装**

1. 安装mysql数据库的客户端与服务端

```
yum install mysql #安装mysql客户端

yum install mysql-server #安装mysql服务端
chkconfig --list|grep mysql # 判断是否安装好
```

2. 启动mysql服务

```
service mysqld start 或者/etc/init.d/mysqld start
/etc/init.d/mysqld status //检查是否启动
```

3. 设置开机启动

```
chkconfig mysqld on
chkconfig --list|grep mysql //检查
```

4. 创建root管理员

```
mysqladmin -uroot password root
```

5. 登录

```
mysql -uroot -proot
```

#### 导入sql文件

```
mysql导入.sql文件

use database;
source sql文件绝对路径;
```

## MINIO部署

注意：minio新版与旧版有些区别，需要加以注意，我这里使用的是新版。

1. 首先安装docker 这里就不作介绍了
2. 执行以下命令 注意 -d表示后台运行

```
docker run -p 9000:9000 -p 9001:9001 --name minio -v /data:/data -e "MINIO_ROOT_USER=user" -e "MINIO_ROOT_PASSWORD=password" minio/minio server /data --console-address ":9001"
```

执行成功的话会看到以下提示（如果是后台运行，请看logs）

![image-20220531212500326](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220531212500326.png)

Minio有一个浏览器界面 对应于console那个 api操作使用的是9000端口

3. 在浏览器中输入地址进入后台

   ![image-20220531212626074](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220531212626074.png)

4. 输入u p后，可以进入管理界面

![image-20220531212717570](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220531212717570.png)

5. 新建一个bucket 文件都是存放在bucket中的 然后注意修改这个bucket的访问权限

![image-20220531212928837](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220531212928837.png)

![image-20220531212912888](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220531212912888.png)

设置完成后，就可以访问到文件了。

比如我们在bucket中放了一张图片 t1.png

则我们可以通过 ip:9000/bucketName/t1.png进行访问

至此，基本使用就完成了，我们也可以使用api来上传下载文件。

## JAVA环境配置

1. 查找java相关列表

```
yum -y list java*
```

2. 安装指定jdk

```
yum install java-1.8.0-openjdk.x86_64
```

3. 完成安装后验证

```
java -version
```

4. 通过yum安装的默认路径为：**/usr/lib/jvm**

5. 将jdk的安装路径加入到JAVA_HOME

```
vi /etc/profile
在文件最后加入：
#set java environment
JAVA_HOME=/usr/lib/jvm/jre-1.6.0-openjdk.x86_64
PATH=$PATH:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME CLASSPATH PATH
```

6. 立即生效

```
. /etc/profile （注意 . 之后应有一个空格）
```


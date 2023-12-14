# WINDOWS11安装WSL2

> 因为这次项目要用到微服务，而且kitex不可以在windows环境下安装，所以方便起见，决定安装一下wsl2。

## 1.启动子系统和虚拟机功能

1. `win+r`输入`appwiz.cpl`
2. 然后点击确定，进入 **程序与功能** 界面，选择 **启用或关闭 Windows 功能**
3. 选择 **适用于 Linux 的 Windows 子系统** 和 **虚拟机平台** 功能
4. 重启电脑！

## 2. 将 WSL2 设置为默认版本

执行以下命令

```
# 更新 wsl
wsl --update
## 将 wsl 版本设置为 wsl2
wsl --set-default-version 2
```

界面如下：

![image-20220519214116737](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220519214116737.png)

## 3. 安装 Linux

MicroSoft Store安装Ubuntu 22.04 也可以选择别的版本

下载后打开，进行安装

![image-20220519215254017](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220519215254017.png)

这里有些语言是乱码不是很懂，选英文正常

![image-20220519215358134](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220519215358134.png)

![image-20220519215729347](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220519215729347.png)

![image-20220519215817264](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220519215817264.png)

安装它说的 更新一下系统

![image-20220519215919899](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220519215919899.png)

到这里其实已经安装好了，我们还需要为其使用图形化界面

`sudo apt install gedit`

![image-20220519221335740](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220519221335740.png)

`sudo apt install neofetch`

然后执行命令 `neofetch`

不过感觉没什么用..

# wsl ubuntu go环境配置

go的环境配置比较简单

`sudo apt install golang`

然后配置一下环境变量

```
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GO111MODULE=on
go env -w GOPATH=yourpath
```

然后将GOPATH下的bin目录加入环境变量（这个不知道是否必须 但是我这边如果要运行kitex好像必须得这么做..因为这个是安装在gopath下的bin里的）

`sudo vim /etc/profile`

按下i

然后把`export PATH=$PATH:/home/renchan/go/bin` 添加到最后一行

按esc 输入:wq退出

然后输入命令`source /etc/profile`

# 安装docker 

wsl上使用docker貌似只需要安装windows版本就好了

总之就是先去官网下载docker desktop win版本 然后点击exe进行安装

https://docs.docker.com/desktop/windows/install/

安装完成后登出账号重启。

然后进入dockerDesktop 的设置 勾选第二个选项，否则不能在wsl中使用docker

![image-20220520091158261](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220520091158261.png)

测试一下wsl是否可能使用docker

![image-20220520091337224](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220520091337224.png)




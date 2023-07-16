# git相关操作

### 绑定ssh

1. 在本地生成ssh-key
           大部分教程会要求填写邮箱号码，但github和gitee都只能获取生成后的那串公钥，并不能破解得到存储于里面的邮箱号数据。这串字符串的功能只是备注，甚至不加也不会影响功能

​		`ssh-keygen -t rsa -C " orange2c "`
​        输入指令后，会要求填写存放路径，以及设置密码，每次连接都需要输入这里设置的密码。我们直接按三次回车就行，存放在默认路径，默认无密码。

2. 复制公钥内容
   进入默认存放密钥的路径

​		linux的默认路径位于 ~/.ssh
​		windows的默认路径位于 /c/Users/本人用户名/.ssh/

​		路径下有两个文件，id_rsa是私钥文件，需保存好。后缀是.pub的文件里就存储着我们要登记到网站的公钥

3将公钥登记到github/gitee

（1）先进入设置  

（2）选择添加新ssh key

（3）把公钥内容贴进去，title随便写，不影响实际连接。确认添加



### 克隆项目

git clone ssh连接

### push操作

`git remote add origin git@github.com:renchandesu/blog_backend.git` 设置远程地址

`git push <远程主机名> <本地分支名>:<远程分支名>`

### pull操作

**git pull** 命令用于从远程获取代码并合并本地的版本。

git fetch 不会自动合并代码  需要手动merge

`git pull <远程主机名> <远程分支名>:<本地分支名>`

### 分支

```
git branch 分支名 创建分支
git checkout -b feature_x 创建并转到
git checkout 分支名  切换分支
git merge 需要合并的分支名 
git push origin 将分支推送到orgin
git branch -d feature_x 删除分支
```


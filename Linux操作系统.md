# Linux操作系统

## 基础篇

### 用户组

### 目录结构

### 常用命令

- 查看网络信息

```
查看网络接口信息：
ifconfig
注意：在新的Linux系统中，ifconfig命令已被废弃，推荐使用ip命令替代。
ip addr

查看路由表信息：
route

查看网络连接状态：
netstat -tuln

查看网络统计信息：
netstat -s

查看DNS服务器信息：
cat /etc/resolv.conf

查看网络接口统计信息：
ifconfig -a

查看网络接口速度和状态：
ethtool <interface_name>

查看某个端口的服务信息 
netstat -tuln | grep <port_number>
sudo lsof -i :23
```

- 查看系统信息

```
uname
```

- 将文件拷贝出linux

```
scp <source_file> <destination_user>@<destination_host>:<destination_path>
scp file.txt remoteuser@192.168.1.100:/home/user/documents
```



### Shell脚本

文件后缀 .sh

#### 运行方式

```
#1.
chmod +x ./test.sh
./test.sh
#2.
bash test.sh
```

#### 语法

##### 变量定义

```bash
name="hillstone" # 注意变量名与等号之间不能有空格
echo ${name} # 使用${name}进行参数的使用 变量名外面的花括号是可选的，加不加都行，加花括号是为了帮助解释器识别变量的边界
# 命令行的参数接收使用 $1 $2进行获取 $0代表文件名
# 交互模式下让用户输入参数 
read -p "提示" x

# 数组变量
arr = (1 2 3)

# 字符串变量
str = ""
# 拼接 注意双引号
str = "hello, "${name}"" 
# 字符串长度
echo ${#str}

# 子字符串 
sub = ${string:1:4}

# 查找字符串 查找字符 i 或 o 的位置(哪个字母先出现就计算哪个)
string="runoob is a great site"
echo `expr index "$string" io`  # 输出 4


# 只读变量
# 使用 readonly 命令可以将变量定义为只读变量，只读变量的值不能被改变。
readonly myUrl
# 删除变量
unset variable_name

```

##### 流程控制语法

```bash
# 如果使用 **((...))** 作为判断语句，大于和小于可以直接使用 **>** 和 **<**。
# [...] 判断语句中大于使用 -gt，小于使用 -lt。
# a=10
# b=20
# if [ $a -gt $b ] | ( ( a>b ) ) 注意空格
# 或者 if [[ || &&  ]]
# then
# 	echo "a>b"
# fi
if con -o或 ！非 -a与
then
#do something
elif con
then
else con
fi



while con
do
#do something
#跳出循环 break contine
done

# for
# in列表是可选的，如果不用它，for循环使用命令行的位置参数。
# 对于循环变量的for循环 可以使用c风格 for (( i=1 ; i<=$n ; i++ )) 
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done

#util
until condition
do
    command
done

# case
# case ... esac 为多选择语句，与其他语言中的 switch ... case 语句类似，是一种多分支选择结构，每个 case 分支用右圆括号开始，用两个分号 ;; 表示 break，即执行结束，跳出整个 case ... esac 语句，esac（就是 case 反过来）作为结束标记。

可以用 case 语句匹配一个值与一个模式，如果匹配成功，执行相匹配的命令。
case 值 in
模式1)
    command1
    command2
    ...
    commandN
    ;;
模式2)
    command1
    command2
    ...
    commandN
    ;;
esac

```

##### 函数

- 可以带function fun() 定义，也可以直接fun() 定义,不带任何参数。
- 参数返回，可以显示加：return 返回，如果不加，将以最后一条命令运行结果，作为返回值。 return后跟数值n(0-255
- 在Shell中，调用函数时可以向其传递参数。在函数体内部，通过 $n 的形式来获取参数的值，例如，$1表示第一个参数，$2表示第二个参数 **注意：第十个参数为${10}**

```
[ function ] funname [()]

{

    action;

    [return int;]

}
```

##### 获取命令的输出

```
var=`ls -a`
var=$(ls -a)
```


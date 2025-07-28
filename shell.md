# shell常用语法

[初步使用](##初步使用)	[Linux中工具链的配置](##Linux中工具链的配置)	[变量](##变量)	[参数](##参数)	[条件判断](##条件判断)	[循环](##循环)	[输入读取](##输入读取)	[函数](##函数)	[正则表达式](##正则表达式)	[文本处理工具](##文本处理工具)

## 初步使用

```sh
#!bin/bash

echo "Hello world!"
echo

# shell
vim helloworld.sh
chmod u+x helloworld.sh

# 在当前bash运行
. helloworld.sh
source helloworld.sh

# 在子bash中运行，无法修改当前shell的变量
./helloworld.sh
```

## Linux中工具链的配置

​	~/.bashrc用于定义当前用户的Bash shell 环境参数。每次打开终端时该文件就会执行。在**~/.bashrc**中添加

```sh
export ARCH=arm
export CROSS_COMPILE=arm-buildroot-linux-gnueabihf-
export PATH=$PATH:/home/book/100ask_imx6ull-sdk/ToolChain/arm-buildroot-linux-gnueabihf_sdk-buildroot/bin
```

- 别名

```bash
alias ll='ls -l'  # 输入 ll 等效于 ls -l
```

## 变量

```sh
# 常用系统变量
echo $HOME; echo $PWD; echo $SHELL; echo $USER

# 定义变量时，=号前后不能有空格
# 全局变量定义，上层shell定义的全局变量下层可以查看修改，但是对上层没有影响
export ARM=arm
```

## 参数

```sh
# 传入的参数个数
$#
# 传入的参数分别为
$0; $1; $2;

# 传入的所有参数，整体和分开
$*; $@;

# 最后一条命令的返回状态
$?
```

- 运算

```sh
A=$[1+2*3]
```

## 条件判断

```sh
if [ $1 -le $b ] 
then
	echo
elif
	echo
fi

case $A in
"1")
	echo
;;
"2")
	echo
;;
*)
	echo
;;
esac

# 常用判断符号
-eq -lt -le -gt -ge -ne
-r -w -x
-e # 文件存在
-f # file
-d # dir
```

## 循环

```sh
for i in $@
do
	echo $i
done
#
for((i=1;i<=100;i++))
do
	sum=$[$sum+$i]
done

while [ $i -le 100 ]
do
	sum=$[$sum+$i]
	i=$[$i+1]
done
```

## 输入读取

```sh
read -t 5 -p "Enter name: " NN
echo NN
```

## 函数

```sh
# 系统函数
basename /home/atguigu/banzhang.txt
# banzhang.txt
basename /home/atguigu/banzhang.txt .txt
# banzhang

dirname /home/atguigu/banzhang.txt
# /home/atguigu

# 自定义函数
function sum()
{
    s=0
    s=$[$1+$2]
    echo "$s"
}
```

## 正则表达式

```sh
# 开头
^
# 结尾
$
# 连续多个字符
a*
# 字符区间
[0-9]* [a-z]*
# 转义
\$
```

## 文本处理工具

```sh
# cut 提取列。-d分隔符；-f第几列
cut -d " " -f 1 file.txt
# awk 提取元素，提取 正则搜索 提取出的所有行的 以 : 分隔的 第七个 元素。
awk -F : '/正则搜索/{print $7}'
```


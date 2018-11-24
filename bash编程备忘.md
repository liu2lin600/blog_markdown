---
title: bash编程备忘
date: 2017-09-14 16:59:34
tags: bash
---

bash编程备忘

<!-- more -->

```
bash -n test.sh 		# 检测语法
bash -x test.sh 		# 调试执行
```

## 说明
> 若从技术的细节来看，shell会依据IFS(Internal Field Seperator) 将命令行所输入的文字拆解为"字段"(word/field)。 然后再针对特殊字符(meta)先作处理，最后重组整行命令行

- `IFS`：有`space`或者`tab`或者`Enter`三者之一组成(我们常用space)
- `CR`: 由`Enter`产生；

`IFS`(Internal Field Seperator)是用来拆解`command line`中每一个词(word)用的，因为`shell command line`是按词来处理的。
而`CR`则是用来结束`command line`用的，这也是为何我们敲`Enter`键，命令就会跑的原因。

除了常用的`IFS`与`CR`, 常用的meta还有：

|meta字符| meta字符作用|
|:--------:|-------------|
|= |设定变量|
|$ | 作变量或运算替换(请不要与`shell prompt`混淆)
|>| 输出重定向(重定向stdout)|
|<|输入重定向(重定向stdin)|
|&|重定向file descriptor或将命令至于后台(bg)运行|
|()|将其内部的命令置于nested subshell执行，或用于运算或变量替换|
|{}|将期内的命令置于non-named function中执行，或用在变量替换的界定范围|
|;|在前一个命令执行结束时，而忽略其返回值，继续执行下一个命令|
|&&|在前一个命令执行结束时，若返回值为true，继续执行下一个命令|
|!|执行histroy列表中的命令|
|... | ...|

## 变量

### 命名规则

- 等号左右两边不能使用分隔符号(IFS),也应避免使用shell的保留元字符(meta charactor)
- 变量的名称(name)不能使用$符号
- 变量的名称(name)的首字符不能是数字(number)
- 变量的名称(name)的长度不可超过256个字符
- 变量的名称(name)及变量的值的大小写是有区别的、敏感的(case sensitive，)

### 特殊变量

```
$0      : 当前脚本的文件名
$#      : 传递给脚本或函数的参数个数
$*      : 传递给脚本或函数的所有参数
$@      : 传递给脚本或函数的所有参数。被双引号包含时，与$*稍有不同
$?      : 上个命令的退出状态，或函数的返回值
$$      : 当前进程的ID
$n      : 传递给脚本或函数的参数。n是一个数字，表示第几个参数。例如，第一个参数是$1
shift N : 替除第N个参数
```

> $\* 和 $@ 都表示传递给函数或脚本的所有参数，不被双引号包含时，都以"$1" "$2" … 的形式输出所有参数，但是当它们被双引号包含时，"$\*" 会将所有的参数作为一个整体，以"$1 $2 ..."的形式输出所有参数；"$@" 会将各个参数分开，以"$1" "$2" … 的形式输出所有参数

### 字符串操作

```
${#str}                    : str的长度
${str:pos}                 : 在str中, 从位置pos开始提取子串
${str:pos:len}             : 在str中, 从位置pos开始提取长度为len的子串
${str#substr}              : 从str的开头, 删除最短匹配substr的子串
${str##substr}             : 从str的开头, 删除最长匹配substr的子串
${str%substr}              : 从str的结尾, 删除最短匹配substr的子串
${str%%substr}             : 从str的结尾, 删除最长匹配substr的子串
${str/substr/replacement}  : 使用replacement, 来代替第一个匹配的substr
${str//substr/replacement} : 使用replacement, 代替所有匹配的substr
${str/#substr/replacement} : 如果str的前缀匹配substr, 那么就用replacement来代替匹配到的substr
${str/%substr/replacement} : 如果str的后缀匹配substr, 那么就用replacement来代替匹配到的substr
```

```
str='/var/log/messages'

echo ${str#*/} 			# var/log/messages
echo ${str##*/} 		# messages
echo ${str%/*} 			# /var/log
echo ${str/s/S} 		# /var/log/meSsages
echo ${str//s/S} 		# /var/log/meSSageS

```

### 变量赋值

```
${arg:-word} ：假如arg为unset或者null，输出word的值但不赋值
${arg:=word} ：假如arg为unset或者null，将word赋值给arg
${arg:?word} ：假如arg为unset或者null，则将word作为错误输出到标准输出
${arg:+word} ：假如arg为unset或者null，则不做展开，返回为空；（刚好与:-相反）

${arg-word}  ：假如arg为unset，输出word的值但不赋值
${arg=word}  ：假如arg为unset，将word赋值给arg
${arg?word}  ：假如arg为unset，则将word作为错误输出到标准输出
${arg+word}  ：假如arg为unset，则不做展开，返回为空；（刚好与-相反）
```

## 表达式

### 算术运算

```
a=11;b=22
let num=$a+$b
let num+=2
num=$[$a*$b]
num=$(($b/$a))

```

### 测试表达式

> `test EXPRESSION`
> `[ EXPRESSION ]`
> `[[ EXPRESSION ]]`：推荐使用

- `-eq`:等于，`-ne`:不等于，`-qt`:大于，`-qe`:大等于，`-lt`:小于，`-le`:小等于
- `==`，`<`，`>`，`!=`，`=~`:左侧字符能否被右侧规则匹配
- `-z $str`: 为空则为真
- `-n $str`: 不空则为真
- `-d file`: 是否是目录
- `-e file`: 存在为真
- `-f file`: 普通文件，`-c`:字符设备，`-p`:管道，`-h`或`-L`:符号链接，`-S`:套接字
- `&&`，`||`，`！`: [ -d /aa ] && [ -f f2 ]
- `-a`，`-o`，`！`: [ -d /aa -a -f f2 ]，[ ! -e /dd ]


```

```

## 判断

1. if...fi



1. if...else...fi



1. if...elif...else...fi


1. case...esac



## 循环

1. for


1. while


1. until



## 函数



## 易错

- echo

```
a="this *.txt"
echo $a  		# this a.txt b.txt ...
echo "$a" 		# this *.txt

b="hello!" 		# 默认情况下bash中的!会被当作执行历史命令
```

- cat

```
cat <<EOF
this is
a 
book
EOF
```

- .exp1.&&.cmd1.||.cmd2.

```
# 由于i++返回值为1(假)，所以i--也会执行
i=0;true && ((i++)) || ((i--));echo $i
```

- for arg in $*

```
# 正确写法
for x in "$@"; do
   echo "parameter: '$x'"
done

# 执行的结果为：
$ ./script 'arg 1' arg2 
parameter: 'arg 1'
parameter: 'arg2'

# 如果"$@"换成$*的结果为：
parameter: 'arg'
parameter: '1'
parameter: 'arg2'
```

- [[ -e file ]]

```
如果file是链接文件，但指向的原文件不存在时，-e的结果为1(假)，如果只想判断是否存在，如下：

[[ -e file || -L file ]]
```

- for循环读文件不靠谱

```
# 建议使用while

while IFS= read -r line; do echo "$line" ; done < file.txt
```

## 简洁

- 位置参数

```
arg=${1:-123} 		# 当$1不存在时，arg=123
```

- 格式化数字

```
printf "%'.2f" 1234455.2323 	# 1,234,455.23
```

- here document(cat <EOF)

```
while read line; do
    CMD...
done << EOF
`grep 1 test.txt`
EOF
```

- 多线程

```
# 使用(CMD)方式fork一个子进程来执行

num=10
function multi_test(){
    echo "working..."
    sleep 2
}

for ((i=0; i < num ;i++)); do
    echo "Fork job $i"
    (multi_test) &
done

wait
echo "All job done!"
```

- vim编辑远程文件

```
vim scp://data01.com//tmp/test.sh 
```

- 编辑当前命令

```
# 遇到输入比较长的命令时，需要修改时，使用 ctrl+x ctrl+e，默认使用emacs，使用vim修改~/.bashrc

export EDITOR='vim'
```




## 参考

- shell 13问
- [bash 编程易犯的错误](http://kodango.com/bash-pitfalls-part-1)






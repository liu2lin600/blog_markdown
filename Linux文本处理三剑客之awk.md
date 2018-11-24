---
title: Linux文本处理三剑客之awk
date: 2016-06-21 13:28:38
categories: Linux基础
tags: [awk,文本处理]
---

Linux文本处理中的屠龙刀出场

<!--more-->

## 简介
awk是一种解释执行的编程语言，用来专门处理文本数据，其名称是由它们设计者的名字缩写而来 ——— Afred Aho, Peter Weinberger 与 Brian Kernighan。常见版本有：

- awk： 最原初的版本，它由 AT&T 实验室开发
- nawk：awk的改进增强版本
- gawk：GNU awk，所有的 GNU/Linux 发行版都包括gawk，且完全兼容awk与nawk

其工作原理为按行读取文件全部或匹配的内容，默认以空格为分隔符将每行分割成字段，可对各字段按某种格式输出

## 语法
`awk [OPTION] '[BEGIN{action}][PATTERN]{action}[END{action}]' file1...`  

说明：

- OPTION：常用选项有`-F`和`-v`等  
- BEGIN{...}：启动时执行的代码部分，且只执行一次，通常用来初始化变量等，可选  
- PATTERN：限定对于输入的行的规则，可选  
- {action}：每一次输入的行都会执行一次主体部分的命令
- END{...}：程序运行结束时执行的代码，可选

## 使用
### 0.内置变量
`$0`：匹配到的行的全部字段  
`$1~$#`：匹配到行分割后的指定字段  
`NF`：number of field，字段个数  
`FS`：input field seperator，输入的分隔符，默认为空白字符  
`OFS`：output field seperator，输出的分隔符，默认为空白字符  
`RS`：input record seperator，输入的换行符  
`ORS`：output record seperator，输出时的换行符  
`NR`：number of record，当前记录的字段数，如果多文件时编号累加  
`FNR`：与`NR`不同的是，`FNR`用于记录正处理的行是当前这一文件中被总共处理的行数  
`FILENAME`：awk命令所处理的文件的名称  
`ARGC`：命令行中参数的个数，其awk命令也算一个参数  
`ARGV`：其是一个数组，保存的是命令行所给定的各参数

```bash
awk '{print NF}' /etc/fstab         # 每行字段数，这里是数量引用，不是对应的值引用
awk '{print $NF}' /etc/fstab        # 每行中的最后一个字段的值
awk '{print NR}' file1 file2        # 会每把所有文档进行总的编号，而不是单独对文件进行编号
awk '{print FNR}' file1 file2       # 会对每个文件的行数进行单独的编号显示
awk '{print FILENAME}' file1        # 显示当前文件名，但会每行显示一次
awk 'END{print FILENAME}' file1     # 显示当前文件名，但只会显示一次
awk 'END{print ARGC}' /etc/fstab    # 显示共有几个参数
```

### 1.OPTION
> `-F`：指明输入时用到的字段分隔符，默认为空白字符   
> `-v var=value`：自定义变量  
> `-f`：读取文件中的awk规则  

```bash
awk -v str='hello' '{print str,$1}' file1       # 
awk 'BEGIN{str="hello"}{print str,$1}' file1    # 效果同上
awk -f file1                                    # awk规则写到脚本中，从中读取
```

### 2.PATTERN
常见的模式类型有地址定界、BEGIN/END、表达式

*地址定界*： 

- `空`：匹配任意输入行
- `/pattern/`：被此模式匹配到的每一行
- `/pat1/,/pat2/`：第一次被模式1匹配的行到第一次被模式2匹配到的行
- `!/patter/`：取反

*表达式*：  

- **x** `>`,`>=`,`<`,`<=`,`==`,`!=`,`~`,`!~` **y**

*BEGIN/END*：

- `BEGIN`：让用户指定在第一条输入记录被处理之前所发生的动作，通常可在这里设置全局变量  
- `END`：让用户在最后一条输入记录被读取之后发生的动作 

```bash
awk '/^UUID/{print $1}' /etc/fstab              # 以UUID开头的行的第一字段
awk -F: '$3>50{print $0}' /etc/passwd           # 以：分割第3字段值大于50的行
awk -F: '$NF~/bash$/{print $0}' /etc/passwd     # 最后字段为bash结尾的行
awk 'BEGIN{print "hello"}{print $1}' file1      # 在最开始输出hello，可以用来制做表头
```

### 3.action

#### print  
`print intem1,item2...`  

> - 各项目之间使用逗号隔开，而输出时则以空白字符分隔
> - 输出的item可以为字符串或数值、当前记录的字段(如$1)、变量或awk的表达式；数值会先转换为字符串后再输出
> - print命令后面的item可以省略，此时其功能相当于print $0, 因此，如果想输出空白行，则需要使用print ""

```bash
awk 'BEGIN{print "line one\nline two\nline three"}'
```

#### printf  
`printf format, item1, item2, ...`

> - 其与print命令的最大不同是，printf需要指定format；
> - format用于指定后面的每个item的输出格式；
> - printf语句不会自动打印换行符；\n

format格式的指示符都以%开头，后跟一个字符；
> `%s`：显示字符串  
> `%d,%i`：十进制整数  
> `%c`：显示字符的ASCII码  
> `%e,%E`：科学计数法显示数值  
> `%f`：显示浮点数  
> `%g,%G`：以科学计数法的格式或浮点数的格式显示数值  
> `%u`：无符号整数  
> `%%`：显示%自身  
>> 修饰符：  
>> `N`: 显示宽度  
>> `-`: 左对齐  
>> `+`：显示数值符号  

```bash
awk -F: '{printf "%-15s %i\n",$1,$3}' /etc/passwd   # 注意加换行符
```

#### 输出重定向  
 
`print items > output-file`  
`print items >> output-file`  
`print items | command`  

特殊文件：  

- `/dev/stdin`：标准输入  
- `/dev/sdtout`：标准输出  
- `/dev/stderr`：错误输出  
- `/dev/fd/N`：某特定文件描述符，如/dev/stdin就相当于/dev/fd/0

```bash
awk -F: '{printf "%-15s %i\n",$1,$3 > "/dev/stderr" }' /etc/passwd
```

#### 操作符
- 算术操作符：**X** `+`, `-`, `*`, `/`, `%`, `**`, `^` **Y**   (**和^同为次方)  
- 赋值操作符：**X** `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `**=`, `^=`, **Y**  `++`,`--`  
- 比较操作符：**X** `>`,`>=`,`<`,`<=`,`==`,`!=`,`~`,`!~` **Y**  
- 逻辑关系符：**X** `&&`,`||` **Y**  
- 三元运算符：selector `?` if-true-exp `:` if-false-exp

#### 条件控制
- **if-else：**  `if (condition) {then-body} else {[ else-body ]}`

```bash
awk -F: '{if ($1=="root") print $1, "Admin"; else print $1, "Common User"}' /etc/passwd
awk -F: '{if ($1=="root") printf "%-15s: %s\n", $1,"Admin"; else printf "%-15s: %s\n", $1, "Common User"}' /etc/passwd
awk -F: -v sum=0 '{if ($3>=500) sum++}END{print sum}' /etc/passwd
```

- **while：**`while (condition){statement1; statment2; ...}`

```bash
awk -F: '{i=1;while (i<=3) {print $i;i++}}' /etc/passwd
awk -F: '{i=1;while (i<=NF) { if (length($i)>=4) {print $i}; i++ }}' /etc/passwd
```

- **do-while：**`do {statement1, statement2, ...} while (condition)`

```bash
awk -F: '{i=1;do {print $i;i++}while(i<=3)}' /etc/passwd
```

- **for：**  
    `for ( variable assignment; condition; iteration process) { statement1, statement2, ...}`  
    `for (i in array) {statement1, statement2, ...}`

```bash
awk -F: '{for(i=1;i<=3;i++) print $i}' /etc/passwd
awk -F: '{for(i=1;i<=NF;i++) { if (length($i)>=4) {print $i}}}' /etc/passwd
awk -F: '$NF!~/^$/{BASH[$NF]++}END{for(A in BASH){printf "%15s:%i\n",A,BASH[A]}}' /etc/passwd
```
 

- **case：**`switch (expression) { case VALUE or /REGEXP/: statement1, statement2,... default: statement1, ...}`

```bash
awk 'BEGIN{foo=0;switch(foo){case 0:print "yes";break; case 1:print "no";break; default: print "unknow"}}'
```

_3.1.3版本添加的功能，需添加`--enable-switch`选项才有效，高版本4+则无需加_

- **next：**  
    提前结束对本行文本的处理，并接着处理下一行

```bash
awk -F: '{if($3%2==0) next;print $1,$3}' /etc/passwd    # 显示其ID号为奇数的用户
```

#### 数组

`array[index]`  

index可以使用任意字符串，当某数据组元素不存在时，要引用其时，awk会自动创建此元素并初始化为空串，因此，要判断某数据组中是否存在某元素，需要使用index in array的方式。

- 遍历：`for (var in array) { statement1, ... }`
- 删除数组变量：`delete  array[index]`  

```bash
netstat -ant | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'      # 每出现一被/^tcp/模式匹配到的行，数组S[$NF]就加1，NF为当前匹配到的行的最后一个字段，此处用其值做为数组S的元素索引；

awk '{counts[$1]++}; END {for(url in counts) print counts[url], url}' /var/log/httpd/access_log     #统计某日志文件中IP地的访问量
```

#### 内置函数
`rand()`：随机数，大于等于0，小于1  
`gsub(str1, str2, str)`：在str中将str1全部替换为str2，支持配符查找  
`index(str1, str2 )`：在str1内找首次出现str2出现的位置，没有则返回0    
`split(str,array[,fieldsep[,seps]])`：将str以fieldsep为分隔符分隔，并将结果保存至array数组中，下标为从0开始  
`length([str])`：返回string字符串中字符的个数  
`substr(str, start [, length])`：截取字符串，从start开始，取length个，start从1开始计数  
`system(command)`：执行系统命令并将结果返回至awk    
`systime()`：系统当前时间戳  
`mktime("YYYY MM DD HH MM SS[ DST]")`：指定日期时间戳，注意加引号  
`strftime(format [,timestamp [,utc-flage]])`：生成指定格式时间，具体格式参考date命令      
`tolower(str)`：转为小写  
`toupper(str)`：转为大写  

*更多函数内容查看man帮助*

```bash
awk 'BEGIN{str="this is a test";split(str,A," ");print length(A);for(i in A){print i,A[i];}}'
awk 'BEGIN{str="str std str2";gsub("str","STR",str);print str}' 
awk 'BEGIN{str="this is a test";b=match(str,"is");print b}'
awk 'BEGIN{str="this is a test2016";print substr(str,4,6)}' 
netstat -ant | awk '/:80\>/{split($5,clients,":");IP[clients[1]]++}END{for(i in IP){print IP[i],i}}' | sort -rn | head -50
awk 'BEGIN{print mktime("2016 06 21 11 22 33")}'
awk 'BEGIN{print strftime("%Y-%m-%d %H:%M:%S",systime())}'
awk '{system("mkdir " ($NF-1)}' filename    # 倒数第二个字段做为目录名创建目录
```
  
以上内容参考自马哥笔记，以备日后查看







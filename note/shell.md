

## 变量

```shell
#定义变量
var=v #如果值不包含任何空白符（例如空格、Tab 缩进等），那么可以不使用引号
var='v' #单引号定义的变量里面的命令不会解析，不解析变量、命令
var="v" #输出时会先解析里面的变量和命令

#命令赋值给变量
var=`command`
var=$(command)

#只读变量
readonly var

#删除变量
unset var
```



## 标准输入输出重定向

```shell
#linux 预留文件描述符 0 - 标准输入 1 - 标准输出 2 - 标准错误
#重定向符 > 覆盖 >> 追加
ls exist.txt 1>stdout.txt #将ls 结果重定向到 stdout.txt中
ls notExist.txt 2>stderr.txt #将ls 标准错误重定向到 stderr.txt中
command 1>stdout.txt 2>stderr.txt #分别重定向
command &>> output.txt #合并追加重定向

/dev/null #特殊设备文件 2>/dev/null 相当于把标准错误丢弃
```



## $值

```shell
#!/bin/sh
set -x #会把每一行执行的内容打印出来
#set -e #遇到标准错误时终止程序
echo $0 #打印当前文件名
set a b c d e f g h i j k
echo $1 ${10} $10  #可以按参数顺序获取参数
echo $# #当前脚本的参数个数
echo $* #当前脚本参数
echo $@ #与$*类似
echo ${11:-233} #获取第11个设置的变量，若无则取默认值233
echo ${12:-233}
```



## string截取

```shell
${string: start :length}	#从 string 字符串的左边第 start 个字符开始，向右截取 length 个字符。
${string: start}	#从 string 字符串的左边第 start 个字符开始截取，直到最后。
${string: -start :length}	#从 string 字符串的右边第 start 个字符开始，向右截取 length 个字符。
${string: -start}	#从 string 字符串的右边第 start 个字符开始截取，直到最后。
${string#*chars}	#从 string 字符串第一次出现 *chars 的位置开始，截取 *chars 右边的所有字符。
${string##*chars}	#从 string 字符串最后一次出现 *chars 的位置开始，截取 *chars 右边的所有字符。
${string%*chars}	#从 string 字符串第一次出现 *chars 的位置开始，截取 *chars 左边的所有字符。
${string%%*chars}	#从 string 字符串最后一次出现 *chars 的位置开始，截取 *chars 左边的所有字符。
```



## 条件判断

### 格式

```shell
#shell 条件判断格式 
if Predicate ;then

elif Predicate ;then

else

fi
```

### test命令

```shell
if test $var -le 2;then
 echo 233
fi

#可以简写为[ expression ] 注意前后空格
if [ $var -le 2 ];then
 echo 233
fi

#在test命令中使用变量时强烈建议用""包起来
```

### 逻辑运算

```shell
if[ Predicate -a Predicate ];then #逻辑与
if[ Predicate -o Predicate ];then #逻辑或
if[ ! Predicate ];then #逻辑非

fi
```

### 文件表达式

```shell
#文件表达式
if[ -e var1 ];then #文件或目录是否存在
if[ -d var1 ];then #是否是目录
if[ -f var1 ];then #是否是常规文件
if[ -L var1 ];then #是否是符号链接
if[ -r var1 ];then #文件或目录是否可读
if[ -w var1 ];then #文件或目录是否可写
if[ -x var1 ];then #文件或目录是否可执行
if[ -s var1 ];then #文件长度不为0
if[ -h var1 ];then #文件是否软链接
if[ var1 -nt var2 ];then #var1是否比var2新
if[ var1 -ot var2 ];then #var1是否比var2旧

fi
```

### 字符串表达式

```shell
if[ s1 = s2 ];then #字符串是否相等
if[ s1 != s2 ];then #字符串是否不相等
if[ -n "s1" ];then #字符串非空判断 要加上""，不然无论如何都会为true
if[ s1 ];then #字符串非空判断，与-n类似
if[ -z s1 ];then #字符串空判断

fi
```

### 算数比较符

```shell
if[ n1 -eq n2 ];then #相等
if[ n1 -ne n2 ];then #不相等
if[ n1 -gt n2 ];then #n1是否大于n2
if[ n1 -ge n2 ];then #n1是否大于等于n2
if[ n1 -lt n2 ];then #n1是否小于n2
if[ n1 -le n2 ];then #n1是否小于等于n2
```



## 数学运算

```shell
#整数运算
((A=2**2)) #幂运算后赋值
A=$((1+1)) #也可以用$获取运算结果
```

```shell
#浮点数运算 bc 
#格式 "option1;option2;...;expression"|bc
#使用scale设置精度
echo "scale=2;2.3*2"|bc
#也可以定义变量
echo "scale=2;pai=3.14;r=3;pai*r*r"|bc
```

```shell
#算数比较
((var > 1))
```



## [[]]

```shell
#[[]]是 Shell 内置关键字，比test更强大
#格式
[[ 条件 ]]

#不需要把变量名用""包围起来，即使变量是空值，也不会出错。
#不能对 >、< 进行转义，转义后会出错。
#支持逻辑运算符不支持 -o -a
#eg:
if[[ ((a > 1)) || ((b > 1)) ]];then
  echo 233
fi

```





## References

## Shell 命令

### 一. Shell 特殊变量

| 变量 | 含义                                                         |
| :--: | :----------------------------------------------------------- |
|  $0  | 当前脚本的文件名                                             |
|  $n  | 传递给脚本或函数的参数。例如： `第一个参数是 $1，第二个参数是 $2` 。 |
|  $#  | 传递给脚本的参数的个数。例如：./run.sh 1 2 3 4  => 4         |
|  $*  | 所有参数。 "$*" 参数被当做一个整体                           |
|  $@  | 所有参数。 "$@"还是遍历每一个参数                            |
|  $?  | 上个命令的退出状态。正确是 0，错误是 1。                     |
|  $$  | 当前Shell进程ID。对于 Shell 脚本，就是这些脚本所在的进程ID。 |

```bash
#!/bin/bash

## Shell特殊变量：Shell $0, $#, $*, $@, $?, $$和命令行参数

echo "\$0 =" $0
echo "\$1 =" $1
echo "\$# =" $#
echo "\$\$ =" $$
echo "\$* =" $*
echo "\"\$*\" =" "$*"
echo "\$@ =" $@
echo "\"\$@\" =" "$@"

echo "print each param from \$*"
for var in $*
do
    echo "$var"
done

echo "print each param from \$@"
for var in $@
do
    echo "$var"
done

echo "print each param from \"\$*\""
for var in "$*"
do
    echo "$var"
done

echo "print each param from \"\$@\""
for var in "$@"
do 
	echo "$var"
done

function func() {
	echo "func \$1 = " $1
	if [ $1 = "11" ]; then
		return 0	## 正确
	else 
		return 1	## 错误
	fi
}
func $2
echo "\$? = " $?
```
执行结果
```bash
dantedeMacBook-Pro:Shell dante$ ./special_var.sh 11 22 33 44
$0 = ./special_var.sh
$1 = 11
$# = 4
$$ = 30161
$* = 11 22 33 44
"$*" = 11 22 33 44
$@ = 11 22 33 44
"$@" = 11 22 33 44
print each param from $*
11
22
33
44
print each param from $@
11
22
33
44
print each param from "$*"
11 22 33 44
print each param from "$@"
11
22
33
44
func $1 =  22
$? =  1
```

### 二. 判断和比较

#### 1. if 参数

- -e filename 如果 filename存在，则为真 （或者 -a）
- -d filename 如果 filename为目录，则为真 
- -f filename 如果 filename为常规文件，则为真 
- -L filename 如果 filename为符号链接，则为真 
- -r filename 如果 filename可读，则为真 
- -w filename 如果 filename可写，则为真 
- -x filename 如果 filename可执行，则为真
- -s filename 如果文件长度不为0，则为真 
- -h filename 如果文件是软链接，则为真
- -o 运行脚本的用户是文件的所有者
- -z string长度为零，则为真
- -n string长度非零，则为真

#### 2. 比较

=  等于  应用于：整型或字符串比较 如果在[] 中，只能是字符串

!=  不等于 应用于：整型或字符串比较 如果在[] 中，只能是字符串

<  小于 应用于：整型比较 在[] 中，不能使用 表示字符串

\>  大于 应用于：整型比较 在[] 中，不能使用 表示字符串

-eq  等于 应用于：整型比较

-ne  不等于 应用于：整型比较

-lt  小于 应用于：整型比较

-gt  大于 应用于：整型比较

-le  小于或等于 应用于：整型比较

-ge  大于或等于 应用于：整型比较

-a  双方都成立（and） 逻辑表达式 –a 逻辑表达式

-o  单方成立（or） 逻辑表达式 –o 逻辑表达式

-z  空字符串

-n  非空字符串

#### 3. [] 和 [[]]

- **[[]] 运算符只是[]运算符的扩充。能够支持<,>符号运算不需要转义符，它还是以字符串比较大小。里面支持逻辑运算符：|| && ，不再使用-a -o**
- **[[]]能用正则，而[]不行**

```bash
 [ "test.php" == *.php ] && echo true || echo false				=> false
 [[ "test.php" == *.php ]] && echo true || echo false			=> true
```

#### 4. 引号

- 单引号

  强引用，它会忽略所有被引起来的字符的特殊处理，被引用起来的字符会被原封不动的使用，唯一需要注意的点是不允许引用自身。

- 双引号

  弱引用，它会对一些被引起来的字符进行特殊处理。

- 反引号

  ``， 是命令替换，命令替换是指Shell可以先执行反引号中的命令，将输出结果暂时保存，在适当的地方输出。

```bash
x="hello"
y=25

echo '$x'
echo '\$x'
echo '"$x"'
echo "$x"
echo "\$x"
echo '{"name":"$x"}'
echo '{"name":"'$x'"}'
echo "{\"name\":\"$y\"}"
echo "{\"name\":"$y"}"

list=`ls -la`
echo '$list'
echo "$list"

## 输出
$x
\$x
"$x"
hello
$x
{"name":"$x"}
{"name":"hello"}
{"name":"25"}
{"name":25}

$list
total 1343160
drwxr-xr-x@   3 dante  staff         96 11  5  2016 $RECYCLE.BIN
drwx------+  20 dante  staff        640  5 22 10:59 .
drwxr-xr-x+ 122 dante  staff       3904  5 20 11:46 ..
```

#### 5. exec 命令

exec 执行程序在父进程中直接执行，exec的执行不会返回以前的shell，而是直接把以前登陆shell作为一个程序看待，在其上进行复制。

```shell
(1) 在当前目录下(包含子目录)，查找所有txt文件并找出含有字符串”bin”的行
find ./ -name "*.txt" -exec grep "bin" {}
(2) 在当前目录下(包含子目录)，删除所有txt文件
find ./ -name "*.txt" -exec rm {}
```

参考：

- http://xstarcd.github.io/wiki/shell/exec_redirect.html
- https://blog.51cto.com/u_5404542/1786753

### 三. Awk实例 

#### 1. 自定义镜像总大小

```bash
## gsub 替换
docker images | grep dante2012 | awk '{print $NF}' | awk '{gsub("MB", "", $0); print $1}' | awk '{sum+=$1} END {print sum"MB"}'
```

### 四. 案例

#### 1. 统计当前目录中以.html结尾的文件总和

```shell
find ./ -name "*.html" -exec du -k {} \; | awk '{sum+=$1} END {print sum"kb"}'
```

#### 2. 批量修改文件名

```shell
## 方式一
for file in $(find ./ -maxdepth 1 -name "*.html"); do
		mv $file kk_${file#*_}
done

## 方式二
## mac
rename 's/aa/bb/' *.html
## linux
rename aa bb *.html
```


---
title: BASH 的一些技巧
description: 阅读 Advanced Bash-Scripting Guide 获得的一些技巧
tags: ["bash", "abs"] 
---

# BASH 的一些技巧

本文记录阅读 [Advanced Bash-Scripting Guide](http://tldp.org/LDP/abs/html/) 时，新获得的一些技巧。

##  Chapter 3. Special Characters

### 进制转换


```bash
# 把制定的进制数转成十进制数
# 2#101011 # 之前的数字是进制数的基数， 101011 是进制数
echo $(( 2#101011 ))    # 43
echo $(( 16#072f3 ))    # 29427
```

### `:` 用于占位符

```bash
# : 用于占位符 
: ${username=`whoami`}
# ${username=`whoami`}   Gives an error without the leading :
#                        unless "username" is a command or builtin...

: ${1?"Usage: $0 ARGUMENT"}     # From "usage-message.sh example script.
```

### `:` 用于评估变量

```bash
: ${HOSTNAME?} ${USER?} ${MAIL?}
#  Prints error message
#+ if one or more of essential environmental variables not set.
```

### `{}` 将代码块的输出保存到文件

这种方式节省了每次输出都要重定向的麻烦，同时也不需要使用全局重定向的模式。方便记录日志。

```bash
{ # Begin code block.
  echo
  echo "Archive Description:"
  rpm -qpi $1       # Query description.
  echo
  echo "Archive Listing:"
  rpm -qpl $1       # Query listing.
  echo
  rpm -i --test $1  # Query whether rpm file can be installed.
  if [ "$?" -eq $SUCCESS ]
  then
    echo "$1 can be installed."
  else
    echo "$1 cannot be installed."
  fi  
  echo              # End code block.
} > "$1.test"       # Redirects output of everything in block to file.

echo "Results of rpm test in file $1.test"
```

## Chapter 4. Introduction to Variables and Parameters

### 获取最后一个命令参数的一些技巧

```bash
args=$#           # Number of args passed.
lastarg=${!args}
# Note: This is an *indirect reference* to $args ...


# Or:       lastarg=${!#}             (Thanks, Chris Monson.)
# This is an *indirect reference* to the $# variable.
# Note that lastarg=${!$#} doesn't work.
```

## Chapter 10. Manipulating Variables

### 字符串截取

```bash
# ${string:position:length} 从 position 开始，截取 length 长度的字符串
stringZ=abcABC123ABCabc
#       0123456789.....
#       0-based indexing.

echo ${stringZ:0}                            # abcABC123ABCabc
echo ${stringZ:1}                            # bcABC123ABCabc
echo ${stringZ:7}                            # 23ABCabc

echo ${stringZ:1:-2}                         # bcABC123ABCa

# 反向截取有意思的地方来了
# Is it possible to index from the right end of the string?
    
echo ${stringZ:-4}                           # abcABC123ABCabc
# Defaults to full string, as in ${parameter:-default}.
# However . . .

echo ${stringZ:(-4)}                         # Cabc 
echo ${stringZ: -4}                          # Cabc
# Now, it works.
# Parentheses or added space "escape" the position parameter.
```

## Chapter 36. Miscellany

### 优化

**禁用 Unicode 支持**

如果不需要 Unicode 支持，可以禁用，这样可以提高一些性能。

```bash
export LC_ALL=C
```

**使用内置命令而不是外部命令**

使用内置的命令比使用外部命令要快很多，因为不需要创建新的进程。

以字符串匹配为例，使用内置的字符串匹配比使用 `grep` 要快很多。

```bash
v="Builtins execute faster and usually do not launch a subshell when invoked."

# Using grep
for i in {1..1000}
do
    echo $v | grep -q  'f*r'
done

# Using BASH pattern matching
for i in {1..1000}
do
    [[ $v == *f*r* ]]
done
exit $?
```

使用 `grep` 匹配的方式执行需要 `3.7` 秒，而后者只需要 `0.008` 秒，两者相差 `462.5` 倍。


**使用`(())` 而不是 `expr` 执行算数运算**

```bash
#!/bin/bash
#  test-execution-time.sh
#  Example by Erik Brandsberg, for testing execution time
#+ of certain operations.
#  Referenced in the "Optimizations" section of "Miscellany" chapter.

count=50000
echo "Math tests"
echo "Math via \$(( ))"
time for (( i=0; i< $count; i++))
do
  result=$(( $i%2 ))
done

echo "Math via *expr*:"
time for (( i=0; i< $count; i++))
do
  result=`expr "$i%2"`
done

echo "Math via *let*:"
time for (( i=0; i< $count; i++))
do
  let result=$i%2
done

echo
echo "Conditional testing tests"

echo "Test via case:"
time for (( i=0; i< $count; i++))
do
  case $(( $i%2 )) in
    0) : ;;
    1) : ;;
  esac
done

echo "Test with if [], no quotes:"
time for (( i=0; i< $count; i++))
do
  if [ $(( $i%2 )) = 0 ]; then
     :
  else
     :
  fi
done

echo "Test with if [], quotes:"
time for (( i=0; i< $count; i++))
do
  if [ "$(( $i%2 ))" = "0" ]; then
     :
  else
     :
  fi
done

echo "Test with if [], using -eq:"
time for (( i=0; i< $count; i++))
do
  if [ $(( $i%2 )) -eq 0 ]; then
     :
  else
     :
  fi
done

exit $?
```

上述脚本执行的结果如下：

```bash
Math tests
Math via $(( ))

real	0m0.130s
user	0m0.130s
sys	0m0.001s
Math via *expr*:

real	2m41.665s
user	1m10.645s
sys	1m36.772s
Math via *let*:

real	0m0.180s
user	0m0.176s
sys	0m0.004s

Conditional testing tests
Test via case:

real	0m0.146s
user	0m0.142s
sys	0m0.005s
Test with if [], no quotes:

real	0m0.186s
user	0m0.184s
sys	0m0.002s
Test with if [], quotes:

real	0m0.201s
user	0m0.196s
sys	0m0.005s
Test with if [], using -eq:

real	0m0.195s
user	0m0.192s
sys	0m0.003s
```

从结果来看，使用 `(( ))` 进行算数运算比使用 `expr` 要快很多，两者相差 `12351.3` 倍。

### 安全问题

**隐藏 shell 脚本源代码**

可以使用 [shc --generic shell script compiler](http://www.datsi.fi.upm.es/~frosal/sources/) 来编译 shell 脚本为二进制可执行文件，这样可以隐藏脚本的源代码。

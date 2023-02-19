---
title: git 不常见的操作技巧
description: 以下记录一些不太常见的git操作技巧
tags: ["git"]
---

# git 不常见的操作技巧

以下记录一些不太常见的git操作技巧

## 删除远程分支或tag

```shell
git push --delete origin tagname|branch_name
```

## 正确的切换到指定的tag

```shell
git checkout -b new_branch tagname
```

## 要修改一系列提交的用户信息

假定从 `a20220a` 开始到当前最新提交的所有用户信息需要修改，可以这样做

```shell
git rebase --onto a20220a \
  --exec "git commit --amend --author=\"wgzhao <wgzhao@gmail.com>\" " a20220a
```

我们可以把这个操作保存为别名，方便以后操作，编辑 `~/.gitconfig`， 增加以下内容：

```ini
[alias]
    reauthor = !bash -c 'git rebase --onto $1 --exec \"git commit --amend --author=$2\" $1' --
```

下次运行就是这样

```shell
git reauthor a20220a "wgzhao <wgzhao@gmail.com>"
```

## git 统计命令

### 统计某人代码提交量

```shell
git log --author="zhaowg" --pretty=tformat: --numstat | \
awk '{ add += $1; subs += $2; loc += $1 - $2 }
 END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -
```

### 统计所有人代码提交量（指定统计提交文件类型）

```shell
git log --format='%aN' | sort -u | \
while read name; do
echo -en "$name\t";
git log --author="$name" --pretty=tformat: --numstat | \
grep "\(.html\|.java\|.xml\|.properties\|.css\|.js\|.txt\)$" | \ 
awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -; 
done
```

### 统计某时间范围内的代码提交量

```shell
git log --author="zhaowg" --since='2019-01-01' --until='2021-02-01' --format='%aN' | sort -u | while read name; do echo -en "$name\t"; git log --author="$name" --pretty=tformat: --numstat | grep "\(.html\|.java\|.xml\|.properties\)$" | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -; done
```

### 查看git提交前5名

```shell
git log --pretty='%aN' | sort | uniq -c | sort -k1 -n -r | head -n 5
```

### 贡献值统计

```shell
git log --pretty='%aN' | sort -u | wc -l
```

### 提交数统计

```shell
git log --oneline | wc -l
```

### 统计或修改的行数

```shell
git log --stat|perl -ne 'END { print $c } $c += $1 if /(\d+) insertions/'
```
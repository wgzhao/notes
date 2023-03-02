---
title: Linux 命令学习
tags: [linux, commands]
---

# Linux 命令学习

这其实很早很早前我在自己博客上贴的内容了，是我 2007 年 1 月开始到当年 7 月，半年多的时间系统学习 Linux 下 `/sbin` 以及 `/usr/sbin/` 的每个命令，这个的缘起还是之前在 2007 年的时候，苦于自己对 Linux 深入学习进入了一个瓶颈，不知道从那个方面可以入手，于是去请教当时公司的 Linux 内核研发经理，应该从哪里方面入手，他回答说，他之前在 Linux 伊甸园（知道这个论坛的估计年级都不小了）上名为 `nntp` 的版主说，一个优秀的 Linux 系统管理员，都应该认真学习`/sbin/` `/usr/sbin` 下的所有命令，你可以从这里入手。

之前是分类记录的，现在我把他整理在一块，这些命令还是基于 RedHat Enterprise 5.0 ，已经很古老了，不过很多命令也很久没有更新了，所以用法一直保留了下来。

## /sbin 下 a 开头的命令

`/sbin` 下 a 开头的命令并不多，大概是这些

> accton adsl-setup adsl-stop arping auditd
> addpart adsl-start agetty asianux-support-check
> adsl-connect adsl-status arp auditctl

其中 adsl 开头的命令占了一半。

分别简单介绍：

### accton

打开或者关闭进程统计（account)，如果不加任何参数，缺省的情况下是关闭进程统计。

要理解这个命令，就需要看看其他几个命令了。
首先我们看看 accton 属于哪个包

```shell
# rpm -qf accton
psacct-6.3.2-35.rhel4
# rpm -qi psacct
....
Description :
The psacct package contains several utilities for monitoring process
activities, including ac, lastcomm, accton and sa. The ac command
displays statistics about how long users have been logged on. The
lastcomm command displays information about previous executed
commands. The accton command turns process accounting on or off. The
sa command summarizes information about previously executed
commmands.
```

从这个描述来看，我们知道 accton 以及 ac,lastcomm,sa 都是用来监控进程的活动性的。

其中 ac 命令显示某个用户已经登录多久了（单位是小时），比如

```shell
# ac root

total 9.13

# ac mlsx

total 0.00
# ac -d
Jan 11 total 0.98
Jan 12 total 0.49
Jan 13 total 1.79
Jan 15 total 3.97
Jan 16 total 0.03
Jan 22 total 1.60
Today total 0.54

# ac –-reboots
total 28.81
```

ac 可以接受的参数的并不是很多，而且也比较容易理解。只是这个 `reboots` 我有点不好理解他，暂时我的理解是本次进入系统于上次进入系统的时间差（小时），也就是最近两次进入系统的时间间隔。
而 `lastcomm` 则显示之前执行的那些命令的信息。

```shell
# lastcomm | head
lastcomm S X root pts/0 0.29 secs Wed Jan 24 21:38
scim-helper-lau SF root __ 4.26 secs Wed Jan 24 21:37
scim-helper-lau S root __ 0.01 secs Wed Jan 24 21:37
ac S root pts/1 0.00 secs Wed Jan 24 21:35
ac S root pts/1 0.00 secs Wed Jan 24 21:35
ac S root pts/1 0.00 secs Wed Jan 24 21:35
ac S root pts/1 0.00 secs Wed Jan 24 21:35
crond SF root __ 0.00 secs Wed Jan 24 21:35
mrtg S root __ 0.83 secs Wed Jan 24 21:35
ac S root pts/1 0.00 secs Wed Jan 24 21:34
```

这里些列分别表示命令的名称，标识，执行帐号，执行终端，已经程序退出的时间。
其中表示的定义如下：

- S: 该命令由超级用户执行。
- F: 命令是在 fork 后执行的，但是接下来没有执行 exec
- C: 命令运行在 PDP－11 兼容模式下（这个向下兼容也太长时间了吧）
- D: 命令中断，且产生了 core 文件。
- X: 命令受到 SIGTERM 信号而中断

而上面的这些信息是否存在就看 accton 的命令操作了，如果他关闭进程信息统计，那么这些命令就无法得到上述信息。
出了运行 accton 命令外，也使用使用

`/etc/init.d/psacct [start|stop]`

开启或者停止进程信息统计。

### adsl-\*

这些命令一看就知道是与 ADSL 有关的。

`adsl-setup` 用来设置对应的信息，采用交互模式，非常的傻瓜化。

`adsl-connect` 是管理一个 PPPoE 连接的命令。

`adsl-start` 开始拨号
`adsl-stop` 断开
`adsl-status` 查看状态

这里 ADSL 的设置和 windows 下有一个小的区别，那就是 windows 是自动获取 DNS 的，但是在 Linux 却不是这样，因此你要先填写 DNS 地址才能正常上网。

`agetty` alternative Linux getty，一个 Linux 下 getty 程序的替换者。一般不手工执行。

`arp`,`arping` 与 ARP 相关的命令，可以清空 ARP 表，刷新 ARP 表，这个程序在高可用的环境下可以得到重用。

`auditctl`,`auditd` 审计程序，按照 RPM 包的说明，他们的主要用处是存储和处理有内核审计子系统产生的审计记录。

## /sbin 下 b 开始的命令

### badblocks

对指定的设备找出坏块,他接受的参数不是太多，你可以指定开始块也结束块。还能用 `-f` 来指定是否需要对查找的分区进行修复，不过在 man 中对 `-f` 这个参数的描述是，
永远都不要使用 `-f`，除非你比这个命令更聪明。

```shell
]# badblocks /dev/hda5
```

man 手册中提到一般不要直接执行 badblocks 命令，而是应该用 `e2fsck` 命令的 `－c` 参数来自动调用 `badblocks` 命令，他会查找坏块并标记起来，以便下次存储数据时，不会存储在这些坏块上。

### blkid

用来定位或者打印块设备属性的命令行工具，他一般和 libuuid(3) 程序库一起使用。他可以推测出一个块设备内容的类型（比如是文件系统，是 swap）和其他属性。

```shell
# blkid /dev/hda5
     /dev/hda5: LABEL="/" UUID="74e3bf86-ce24-411d-ac35-c4bc49964f36" SEC_TYPE="ext3" TYPE="ext2"
# blkid /dev/hda1 /dev/hda1: UUID="6C2B-F039" TYPE="vfat"
```

UUID 在 RAID 和 LVM 中有比较广泛的使用，在 RAID 中，寻找某一个设备或者分区不是根据你的设备号（比如 hda3），也不是标签（比如 `/HOME`)，因为手工创建的分区默认没有标签，而是根据 UUID 号，比如你的第一个 SCSI 设备因为槽位的变化，那么在 linux 中设备好也发生了变化（比如从 sda 变成 sdb)，但是系统依然能够识别出来。

而槽位的改变在服务器上很容易发生，因为大部分都是支持热插拔的。在 LVM 的配置文件中，你更能非常清楚的看到 UUID 号。

### blockdev

在命令行调用设备的 `ioctl` 函数。在 Linux 系统中，似乎对设备的直接操作只有 `ioctl` 函数了。他接受的参数不是太多，而且都是一一对应的。

```shell
--setro 设置设备为只读
--getro 读取设备是否为只读（成功为 1，0 则为可读写）
--setrw 设置设别为可读写
--getss 打印设备的扇区大小，通常是 512
--getsize 打印设别的容量，按照一个扇区 512 个字节计算
--setra N Set readahead to N 512-byte sectors.
--getra 打印 readahead
--flushbufs 刷新缓冲
--rereadpt 重读分区表。
```

个人觉得 `--setro`, `--setrw` 比较有用，这个 `mount -o ro(rw)`是有区别的，mount 是在文件系统这个级别上对某个分区挂载为只读或可读写。

而 blockdev 则是在设别这个级别上设置为只读和可读写。

看下面的命令输出结果就一目了然了。

```shell
# blockdev –setro /dev/hda4
# blockdev –getro /dev/hda4
    1
# mount /dev/hda4 /misc -o rw
    mount: block device /dev/hda4 is write-protected, mounting read-only
# umount /dev/hda4
# blockdev –setrw /dev/hda4
# blockdev –getro /dev/hda4
    0
# mount /dev/hda4 /misc -o rw
# touch /misc/one
# umount /dev/hda4
# mount /dev/hda4 /misc -o ro
# rm -f /misc/one
    rm: 无法删除‘/misc/one’: 只读文件系统
```

这个命令在高可用环境比较实用。比如红旗的 HA 产品就使用了类似的命令来防止，红旗 HA 停止时，会自动将他监管的设备设置为只读，防止用户的改写，除非你手工设置为可读写（红旗 HA 使用了 `clproset` 命令，其作用应该和这个差不多）

## /sbin 下 c 开始的命令

/sbin 目录下 c 开头的命令也没有多少，列出如下

> cardctl cardmgr change_console chkconfig clock consoletype cryptsetup ctrlaltdel

其中 `clock` 还是软连接。

```shell
# ls -l clock
lrwxrwxrwx 1 root root 7 9月 7 12:07 clock -> hwclock
```

`cardctl` 和 `cardmgr` 是对 PCMCIA 设备的操作，不过似乎直接使用这两个命令来管理 PCMCIA 设别的机会应该很少。一来 PCMCIA 插槽只在笔记本上有，二来目前 Linux 下能支持的 PCMCIA 设备还真不多，似乎除了一些无线网卡和很少的无线上网卡支持外，其他的我就不清楚了，所以这两个命令我没用过，看了 man 手册，也没有看出名堂，忽略之。

### change_console

改变终端，这个命令是 init 来用的，我怎么加参数来执行，也没有看出什么变化，man 手册说的很少，忽略之。

### chkconfig

更新和查询系统服务的运行级别信息。我们知道大部分的 Linux 发行版本都有 0-6 共 7 个级别的运行模式，先抛开关机的 0 级别和重启的 6 级别，那么就还有 1－5 共 5 个运行级别，Linux 系统允许你定义在不动的运行级别默认启动不动的系统服务。

而实际上所有的执行脚本都是在`/etc/rc.d/init.d/`下面。

而每个不同级别的可以运行的系统服务定义在`/etc/rc.d/rc[0-6].d`里，里面只有软连接。软连接分成了两种类型，一种是 K 开始的软连接，一种是 S 开头的软连接。分别表示需要停止的服务和启动的服务。

其中启动顺序有连接名中的数字决定，数字越小，启动的优先级就越高。

很显然如果要我们手工来管理这些连接，恐怕表示一件容易的事情。也是 `chkconfig` 就出现了。

目前 Linux 下的 `chkconfig` 主要完成 5 个方面的功能：

1. 增加可以管理的系统服务
2. 移走可以管理的系统服务
3. 列出当前服务器的启动信息
4. 改变系统服务的启动信息
5. 检查特定服务的启动状态

我们举一个例子阐述上面的 5 个功能

假设我们写了一个服务程序，我们希望像其他系统服务那样可以定义不同级别的启动停止方式，而不是简单的加入到 `/etc/rc.d/rc.local` 文件中。那么我们首先要创建一个运行脚本放到 `/etc/rc.d/init.d` 目录下，并设置好权限。如果你不知道如何修改，那么你可以考参 `init.d` 目录下的任何一个运行脚本。下面的是我的一个最简单的脚本

```shell
# cat /etc/rc.d/init.d/test
#! /bin/sh
#
# test simple test system service for management by chkconfig
#

# Source function library.
. /etc/rc.d/init.d/functions

# Set defaults

start(){
	echo -n “Starting test: ”
	/usr/local/bin/tmy start
	RETAL=$?
	echo
}

stop(){
	echo -n “Stopping test: ”
	/usr/local/bin/tmy stop
	RETVAL=$?
	echo
}

restart(){
	stop
	start
}

# See how we were called.
case “$1″ in
	start)
	start
	;;
	stop)
	stop
	;;
*)
echo “Usage: test {start|stop|}”
RETVAL=1
esac

exit $RETVAL
```

```shell
# cat /usr/local/bin/tmy
#!/bin/bash
if [ $1 = “start” ];then
echo “have started my services”
elif [ $1 = “stop” ]; then
echo “have stoped myservices”
fi
exit 0
```

现在你可以使用`/etc/init.d/test start|stop`的方式启动这个服务。但是

` chkconfig --list test test` 服务不支持 chkconfig

所以，接下来我们要把这个服务器加入到可管理系统服务中，我们需要在执行脚本中加入两行让 chkconfig 识别的注释

```shell
# chkconfig:2345 20 80
# description: test simple test system service for management by chkconfig
```

这两行注释是告诉 `chkconfig` 命令，这脚本应该在 2，3，4，5 这 4 个级别启动，且启动优先级是 20，而对应的停止优先级是 80。
加入后，我们再试试 chkconfig 命令

```shell
# chkconfig --list test
test 服务支持 chkconfig，但它在任何级别中都没有被引用(运行“chkconfig --add test”)
```

恩，至少 chkconfig 认识出了 test，而且也说了可以被管理，只是没有加入。

根据提示，我们需要加入他

```shell
# chkconfig --add test
# echo $?
0
# chkconfig --list test test
	0:关闭 1:关闭 2:启用 3:启用 4:启用 5:启用 6:关闭
```

这和我们期望的是一样的。然后我们看看 test 软连接

```shell
# ls -l /etc/rc.d/rc6.d/K80test
lrwxrwxrwx 1 root root 14 1月 27 20:47 /etc/rc.d/rc6.d/K80test -> ../init.d/test
# ls -l /etc/rc.d/rc3.d/S20test
lrwxrwxrwx 1 root root 14 1月 27 20:47 /etc/rc.d/rc3.d/S20test -> ../init.d/test
```

到此，test 服务已经加入了可被 chkconfig 管理的系统服务范围内。那么我们事实修改和删除看看。

```shell
# chkconfig --list test test
0:关闭 1:关闭 2:启用 3:启用
4:启用 5:启用 6:关闭
# chkconfig --level 3 test off
# chkconfig --list test test
0:关闭 1:关闭 2:启用 3:关闭 	4:启用 5:启用 6:关闭
# ls -l /etc/rc.d/rc3.d/K80test
lrwxrwxrwx 1 root root 14 1月 27 20:54 /etc/rc.d/rc3.d/K80test ->	../init.d/test
# chkconfig --del test
# chkconfig --list test
test 服务支持 chkconfig，但它在任何级别中都没有被引用(运行“chkconfig --add test”)
```

如果要彻底删除 test 服务，那么就只要删除 test 执行脚本就行了。

### consoletype

打印连接到标注输入的终端类型；这是 Asianux 特有的命令，他根据终端的不同类型打印出不同的信息。

如果是虚拟终端(/dev/tty\* 或者不是串行终端的`/dev/console`)，打印出 vt

如果标准输入是串行终端(`/dev/console` 和或者`/dev/ttyS*`)，打印 serial

如果标准输入是伪终端（`/dev/pts/\*`)，打印 pty

```shell
# who
root pts/0 Jan 27 19:41 (:0.0)

# consoletype
pty

# who
root tty1 Jan 27 21:05

# consoletype
vt

# cat test.c
#include
#include
int main()
{
	int fd1,fd2;
	fd1=open(“test.sh”,O_RDONLY);
	if (!fd1)
	{
	perror(“error open test.sh”);
	exit(65);
}
fd2=dup2(fd1,0);
system(“/sbin/consoletype >/tmp/a”);
return 0;
}
# gcc test.c -o test
# ./test
# cat a
serial
# consoletype
pty
```

### cryptsetup

对指定的设别进行加密，这里的设别必须是 `/dev/mapper` 下的设备，那这样看来 2.4 的内核是没有这个命令了。
他对应的应该有另外一个命令 `dm-crypt`，但是我的系统上找不到。
该命令没有 man 手册，其 help 帮助如下：

```
# cryptsetup –help
Usage: cryptsetup [OPTION…] []
-v, –verbose Shows more detailed error messages
-c, –cipher=STRING The cipher used to encrypt the disk (see
/proc/crypto) (default: “aes”)
-h, –hash=STRING The hash used to create the encryption key from
the passphrase (default: “ripemd160″)
-y, –verify-passphrase Verifies the passphrase by asking for it twice
-d, –key-file=STRING Read the key from a file (can be /dev/random)
-s, –key-size=BITS The size of the encryption key (default: 256)
-b, –size=SECTORS The size of the device
-o, –offset=SECTORS The start offset in the backend device
-p, –skip=SECTORS How many sectors of the encrypted data to skip
at the beginning

Help options:
-?, –help Show this help message
–usage Show this help message

is one of:
create – create device
remove – remove device
reload – modify active device
resize – resize active device
status – show device status
is the device to create under /dev/mapper
is the encrypted device
```

这个 help 似乎也不能帮我们了解更多的内容。只好作测试了。

我用了一个 U 盘来创建 LVM，这样在` /dev/mapper` 下面就有对应的设备了。

```shell
# ls -l /dev/mapper/vg0-lv0
    brw-rw—- 1 root disk 253, 0 1月 27 22:13 /dev/mapper/vg0-lv0

# mount /dev/mapper/vg0-lv0 /misc

# ls -l /misc
    总用量 12
    drwx—— 2 root root 12288 1月 27 22:21 lost+found

# touch /misc/one

# ls -l /misc
    总用量 12
    drwx—— 2 root root 12288 1月 27 22:21 lost+found
    -rw-r–r– 1 root root 0 1月 27 22:22 one

# umount /misc

# cryptsetup create cytest /dev/mapper/vg0-lv0
    Enter passphrase:mypassword

# ls -l /dev/mapper/vg0-lv0
    brw-rw—- 1 root disk 253, 0 1月 27 22:13 /dev/mapper/vg0-lv0

# mount /dev/mapper/vg0-lv0 /misc
    mount: /dev/mapper/vg0-lv0 already mounted or /misc busy

# cryptsetup remove cytest

# mount /dev/mapper/vg0-lv0 /misc

# ls -l /misc
    总用量 12
    drwx—— 2 root root 12288 1月 27 22:21 lost+found
    -rw-r–r– 1 root root 0 1月 27 22:22 one
```

加密前的设备和加密后的设备我用 `od` 命令查看了输出值，发现没有不同，看了他不是根据内容加密的，具体情况需要查[官方网站](http://www.saout.de/misc/dm-crypt/)才知道了。

### ctrlaltdel

设置 `Ctrl-Alt-Del` 组合键的功能，这里有两种方式 hard 和 soft。

hard 表示当接受 `Ctrl-Alt-Del` 组合键时，系统立刻重启计算机，而不会调用 `sync(2)`命令，也不会作其他一些准备，这有点类似不延时的 reset 按钮。

而 soft 则表示 `Ctrl-Alt-Del` 组合键按下时，他发送 SIGINT 信号给 init 进程，那剩下的事情就是看 init 如何做了。

## /sbin 下 d 开头的命令

命令如下

> debugfs delpart dhclient-script dmsetup dump dump.static
> debugfs.ocfs2 depmod dhcp6c dmsetup.static dump_cis
> debugreiserfs dhclient dmraid dosfsck dumpe2fs

其中 `dump.static` 软连接到 dump 命令，而 dump 程序是一个静态编译的命令。

### debugfs

debugfs, debufs.ocfs2, debugreiserfs 是文件系统调试工具，其中 debugfs 我使用过一次，其目的是希望恢复删除的某个文件。

但是实验表明，ext2 的文件系统有恢复的可能性，但是 ext3 却没有可能。在 ext3 文件系统官方站点的 FAQ 中，就有一个这样的问题，其中答案就是告诉你，采用 `rm -f` 方式删除的命令是没有办法恢复的，因为他不仅仅只是作删除的标记。

而 `ocfs2` 目前并不稳定，这是 Oracle 推出的一种文件系统，是为 Oracle 数据库的高可用服务的，但是在实际的项目中，ocfs 表现的并如人意。起文件系统都用得少，那么调试就更少了。

`reiserfs` 是 SuSe 发行版本中默认的文件系统格式，RedHat 缺省是 ext2/ext3 的文件系统。 所以 debugreiserfs 也就没有用过了，加上 reiserfs 的开发者因为谋杀妻子的罪名被逮捕，因此这中文件系统前途还不知道怎么样。

### depart

`delpart` 这 个命令很奇怪，没有 man 手册，没有 info 信息，帮助也是少得可怜，似乎很不符合 `*nix` 下程序开发的习惯。而他所属的 rpm 包又是 `util-linux`，这是一个工具集，因此 `rpm -qi` 也不能找到有用的信息。那就只能从帮助和起程序命令来看了。

刚看到这个名字，我以为他是 `parted` 程序的一部分，结果不是。看帮助

```shell
# delpart  –help
usage: delpart diskdevice partitionnr
```

似乎也应该是和分区有关系。我的猜想是他应该是删除一个指定的分区。下面是我的实验过程

首先把我的 U 盘分一个分区出来

```shell
# fdisk -l /dev/sda

Disk /dev/sda: 252 MB, 252968960 bytes
8 heads, 61 sectors/track, 1012 cylinders
Units = cylinders of 488 * 512 = 249856 bytes

Device Boot Start End Blocks Id System
/dev/sda1 1 81 19733+ 83 Linux

# delpart /dev/sda 1

# fdisk -l /dev/sda

Disk /dev/sda: 252 MB, 252968960 bytes
8 heads, 61 sectors/track, 1012 cylinders
Units = cylinders of 488 * 512 = 249856 bytes

Device Boot Start End Blocks Id System
/dev/sda1 1 81 19733+ 83 Linux

# delpart /dev/sda 1
BLKPG: No such device or address
```

从实验来看，他应该是删除一个分区的，而且我们从 delpart 程序反馈信息来看，他认为他删除了 sda1，但是 fdisk 命令却认为这个分区还在。
接下来我测试这个分区是否能 mkfs

```shell
# mkfs.ext2 /dev/sda1
mke2fs 1.35 (28-Feb-2004)
Could not stat /dev/sda1 — 没有那个文件或目录

The device apparently does not exist; did you specify it correctly?
```

看来真的已经删除了，但是为什么 fdisk 不这么认为呢？而且 parted 也认为这个分区是存在的

```shell
# parted /dev/sda
GNU Parted 1.6.19
Copyright (C) 1998 – 2004 Free Software Foundation, Inc.
This program is free software, covered by the GNU General Public License.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without
even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU
General Public License for more details.

使用 /dev/sda
(parted) print
/dev/sda 的磁盘几何结构：0.000-241.250 兆字节
磁盘标签类型：msdos
Minor 起始点 终止点 类型 文件系统 标志
1 0.030 19.300 主分区
```

但是内核却认为这个分区已经不存在了

```shell
# cat /proc/partitions
major minor #blocks name

3 0 29302560 hda
3 1 5246608 hda1
3 2 5178600 hda2
3 3 1 hda3
3 4 13267768 hda4
3 5 5606653 hda5
8 0 247040 sda
```

而使用 sfdisk 命令同样也认为 sda1 存在，但是在扫描 sda1 时停止

```shell
# sfdisk /dev/sda
Checking that no-one is using this disk right now …
OK

Disk /dev/sda: 1012 cylinders, 8 heads, 61 sectors/track
Old situation:
Units = cylinders of 249856 bytes, blocks of 1024 bytes, counting from 0

Device Boot Start End #cyls #blocks Id System
/dev/sda1 0+ 80 81- 19733+ 83 Linux
/dev/sda2 0 – 0 0 0 Empty
/dev/sda3 0 – 0 0 0 Empty
/dev/sda4 0 – 0 0 0 Empty
Input in the following format; absent fields get a default value.

Usually you only need to specify and (and perhaps ).

/dev/sda1 :
```

后面是 `Ctrl+C` 退出的。

strace 了一下 fdisk 程序，没有找到原因。这个问题也暂时留下了。

### depmod

为 modprobe 工具提供一个正确的模块依赖性列表（modules.dep)。 这个命令对于新加入的模块能够像已经有的模块那样添加和删除起到了很好的作用。

我们知道在 Linux 中，内核模块是是在`/lib/modules/ `下面的。使用 `insmod/modprobe` 默认也是从这个目录去找你要加载的模块，除非你用 `-f` 来指定模块的路径。

而对于编译的内核模块，如果希望 modprobe 程序能够仅仅提供模块名就可以加载的话，那么至少要做两件事情。

1. 将新的模块复制到正确的位置。
2. 使用`depmod -A` 或者`depmod -a`来生产模块依赖性比列表。

其中 `－a` 是表示所有模块都扫描一次，然后生成列表，而 `－A` 表示进扫描新增的或者更新的模块，然后更新模块依赖性列表。

这两个参数是最常用的了。我目前也就仅仅用过这两个参数。

刚才看了 man 手册，还有一些参数，比如 `－b` 表示模块的基本路径在哪里，利用这个参数你可以将模块从 `/lib/modules/` 中迁移出来。而 `-n` 参数表示仅仅只是打印出模块依赖性列表，而不写入，相当于仅仅是演示，而不会产生实际效果。

### dhclient

dhclient， dhclient-script, dhcp6 这是 DHCP 协议中用的，一般我们都不会直接去执行这几个命令，而是配置他的配置文件和运行 `/etc/rc.d/init.d/dhcpd` 命令。所以这里也不深入研究了。

### dmraid

device-Mapper 软 RAID 工具，他主要是用来执行设备发现，RAID 激活和 ATARAID 属性显示。因此我们需要和 RAID 制作工具一起来讲。

### dmsetup

底层逻辑卷管理(LVM)，这是 device-mapper 包中带的唯一一个可执行程序，除了那个静态版的 `dmsetup.static`。这个命令我想留到和 `device-mapper-multipath` 一起来研究，这里就不阐述了。

### dosfsck

检查和修复 MSDOS 文件系统，Linux 还真的有意思，居然来一个这样的工具，不过我从未在 Linux 下修复过 MS 的文件系统，总是感觉有点玄。

而且我觉得也没有机会。服务器上吧，MS 的文件系统和 Linux 的文件系统是不会存在一个硬盘上的，服务器上需要双系统吗？自己的机器上，如果有 MS 的文件系统，那么就应该有其操作系统，那么自己的工具修复自己的东西就好了。当然也不排除这种可能性：以前是有双系统，而一些数据是放在 MS 的文件系统上的，后来觉得 MS 的系统不爽，于是咔嚓掉了。

但是因为数据太多，其他分区也就不懂了，于是就出现了没有 MS 的系统，但是有 MS 的分区。这个时候是不是 dosfsck 能派上用场呢？

### dump

ext2/ext3 文件系统备份。dump 程序检查 ext2/3 文件系统上的文件，然后猜测那些文件需要备份。这些文件被复制到指定的磁盘，磁带或者其他存储介质上。他还可以作远程的备份。

如果 dump 出来的数据大于指定的备份存储介质，他将分卷。dump 支持的参数比较多。

- `-level#` dump 级别，0 是全备份，0 以上的数字是增量备份。历史原因，dump 程序一般只使用 0-9 这几个数字，不过现在的 dump 版本支持 9 以上的数字（但是，不同的数字意味着什么，man 手册中并没有说）
- `-a` “自动大小(auto-size)”. 忽略所有磁带的长度计算。他将一直写到磁带结束，对于大部分现在的磁带机，这中工作方式是最佳的，也是缺省模式。这种方式特别是在增加数据到一个磁带上或者具有硬件压缩功能的磁带机效果更好。
- `-A` 归档文件，在指定的归档文件上创建一个 dump 内容表(table-of-contents)，将来可以被 restore(8)程序用来判断一个正准备回复的文件是否在 dump 文件中.
- `-b` 块大小.每个 dump 记录的 KB 数.缺省是 10.最大值是 1024,不过与你的内核限制有关系.
- `-B` 记录数,每卷的快数(每块 1KB).当 dump 程序发现到了存储介质的结尾时,他会要求你更换卷.
- `-D` 文件.设置存储之前先前完全或者增量 dump 信息的文件路径.
- `-e` inode 列表,不准备 dump 的 inode 列表,多个 inode 之间用逗号分隔,你可以使用 stat(1)命令来找到文件或者目录的 inode 值.
- `-E` 文件,从文件中排除不 dump 的 inode,文件格式采用一行一个 inode,而且应该是排序的.
- `-f` 文件.写备份到文件;这可以是特定的设备文件,比如/dev/st0,/dev/fd0,/dev/sdc,或者是一个普通文件,或者是标准输出(`-`)， 多个文件采用逗号分隔.每一个文件将存储一个 dump 卷.如果文件命采用 host:file 或者 user@host:file 格式,dump 将使用 rmt(8)程序将数据写入到远程机器指定的文件上(改文件应该存在,dump 不会使用创建一个远程- `文件).
- `-F` 脚本,在每一个磁带的最后执行这个脚本(除了最后一盘磁带)
- `-L` 标签,使用用户提供的字符串作为标签写入到 dump 都,这样一来,像 restore(8)和 file(8)程序就可以访问它.标签的最大长度不能超过 LBLSIZE(当前是 16)个字符,而且字符串必须是 `\0` 结尾.
- `-S` 大小评估。用来计算 dump 需要的空间，但是并不真正执行 dump 操作。这对增量备份有好处，因为可以知道需要多少存储介质。
- `-T` 时间。使用指定的时间来作为 dump 的时间，而不是从 `/etc/dumpdates` 里读取。这里时间的格式和 ctime(3) 的格式相同，遵守 RFC822 时区规范。他与 `-u` 参数是互斥的。
- `-u` 成功 dump 后，更新 `/etc/dumpdates` 文件。/etc/dumpdates 文件格式是人可以阅读的，每行记录一个时间，采取了叫宽松的格式。
- `-y` 写入磁带时压缩块，采用 lzo 链接库。
- `-z` 压缩级别，采用 zlib 库，这个参数仅在将数据 dump 到文件，管道或者磁带机时才有用，缺省值是 2.注意指定这个值的时候，参数和值之间应该没有参数

下面是一个压缩到文件的简单例子

```shell
# dump -0 -f /tmp/rootdump /root
DUMP: Date of this level 0 dump: Sun Jan 28 22:06:46 2007
DUMP: Dumping /dev/hda5 (/ (dir root)) to /tmp/rootdump
DUMP: Label: /
DUMP: Writing 10 Kilobyte records
DUMP: mapping (Pass I) [regular files]
DUMP: mapping (Pass II) [directories]
DUMP: estimated 15509 blocks.
DUMP: Volume 1 started with block 1 at: Sun Jan 28 22:06:48 2007
DUMP: dumping (Pass III) [directories]
DUMP: ACLs in inode #2 won't be dumped
DUMP: ACLs in inode #187672 won't be dumped
DUMP: ACLs in inode #204060 won't be dumped
DUMP: ACLs in inode #674281 won't be dumped
DUMP: dumping (Pass IV) [regular files]
DUMP: ACLs in inode #505922 won't be dumped
DUMP: ACLs in inode #505923 won't be dumped
DUMP: ACLs in inode #505924 won't be dumped
DUMP: ACLs in inode #505925 won't be dumped
DUMP: ACLs in inode #505926 won't be dumped
DUMP: ACLs in inode #505927 won't be dumped
...
DUMP: Closing /tmp/rootdump
DUMP: Volume 1 completed at: Sun Jan 28 22:06:59 2007
DUMP: Volume 1 15340 blocks (14.98MB)
DUMP: Volume 1 took 0:00:11
DUMP: Volume 1 transfer rate: 1394 kB/s
DUMP: 15340 blocks (14.98MB) on 1 volume(s)
DUMP: finished in 11 seconds, throughput 1394 kBytes/sec
DUMP: Date of this level 0 dump: Sun Jan 28 22:06:46 2007
DUMP: Date this dump completed: Sun Jan 28 22:06:59 2007
DUMP: Average transfer rate: 1394 kB/s
DUMP: DUMP IS DONE
```

我们来看看 dump 出来的文件是有什么特征

```shell
# file /tmp/rootdump
/tmp/rootdump: new-fs dump file (little endian), This dump Sun Jan 28 22:06:46 2007, Previous dump Thu Jan 1 08:00:00 1970, Volume 1, Level zero, type: tape header, Label /, Filesystem / (dir root), Device /dev/hda5, Host lancy, Flags 3

# du -sh /tmp/rootdump
15M /tmp/rootdump

# du -sh /root
15M /root
```

在不指定 `-z` 参数时，dump 是没有压缩效果的。

再看看压缩效果

```shell
# dump -0 -z9 -f /tmp/rootdump.z9 /root
......
#ls -h /tmp/rootdump.z9
-rw-r--r-- 1 root root 7.4M
Jan 30 20:56 /tmp/rootdump.z9
```

看上去效果还不错

### dump_cis

显示 PCMCIA 设别的信息。因为我的无线上网卡不这次，因此他仅仅给出了下面的信息

```shell
# dump_cis no pcmcia driver in /proc/devices
```

### dumpe2fs

导出 ext2/ext3 文件系统信息，显然这也是一个重量级的工具，在文件系统出现损坏的情况下用得着。

他打印出文件系统的超级块和块组信息。我们先看一个具体的例子，下面是对我刚创建个一个分区的 dump 信息。

```shell
# dumpe2fs /dev/sda1
dumpe2fs 1.35 (28-Feb-2004)
Filesystem volume name:
Last mounted on:
Filesystem UUID: f74b41c6-637f-436f-ab9c-e1cbd532abf8
Filesystem magic number: 0xEF53
Filesystem revision #: 1 (dynamic)
Filesystem features: has_journal resize_inode filetype sparse_super
Default mount options: (none)
Filesystem state: clean
Errors behavior: Continue
Filesystem OS type: Linux
Inode count: 4944
Block count: 19732
Reserved block count: 986
Free blocks: 17905
Free inodes: 4933
First block: 1
Block size: 1024
Fragment size: 1024
Reserved GDT blocks: 77
Blocks per group: 8192
Fragments per group: 8192
Inodes per group: 1648
Inode blocks per group: 206
Filesystem created: Sun Jan 28 21:43:19 2007
Last mount time: n/a
Last write time: Sun Jan 28 21:43:19 2007
Mount count: 0
Maximum mount count: 26
Last checked: Sun Jan 28 21:43:19 2007
Check interval: 15552000 (6 months)
Next check after: Fri Jul 27 21:43:19 2007
Reserved blocks uid: 0 (user root)
Reserved blocks gid: 0 (group root)
First inode: 11
Inode size: 128
Journal inode: 8
Default directory hash: tea
Directory Hash Seed: d1aef805-54e4-4d41-90f7-e509a85a7028
Journal backup: inode blocks

Group 0: (Blocks 1-8192)
Primary superblock at 1, Group descriptors at 2-2
Block bitmap at 80 (+79), Inode bitmap at 81 (+80)
Inode table at 82-287 (+81)
6861 free blocks, 1637 free inodes, 2 directories
Free blocks: 1332-8192
Free inodes: 12-1648
Group 1: (Blocks 8193-16384)
Backup superblock at 8193, Group descriptors at 8194-8194
Block bitmap at 8272 (+79), Inode bitmap at 8273 (+80)
Inode table at 8274-8479 (+81)
7905 free blocks, 1648 free inodes, 0 directories
Free blocks: 8480-16384
Free inodes: 1649-3296
Group 2: (Blocks 16385-19731)
Block bitmap at 16385 (+0), Inode bitmap at 16386 (+1)
Inode table at 16387-16592 (+2)
3139 free blocks, 1648 free inodes, 0 directories
Free blocks: 16593-19731
Free inodes: 3297-4944
```

从这个 dump 信息我们能够得到几个很重要的信息，无论对于你将来的正在备份，恢复还是文件系统修复都是至关重要的。

```
Free blocks: 17905 //能够存储数据的块大小
Free inodes: 4933 //能够使用的 i 节点 Block
size: 1024 //块大小
Primary superblock at 1，Backup superblock at 8193 //主超级块以及备份超级块的位置
```

free blocks 和 block size 决定了存储数据的空间大小，同时 block size 在某些备份软件中是需要指定的。
而 free inode 表说明能够创建的文件或目录总和。

因为一个文件或目录需要占用一个 inode 节点。因此即使你的空间还足够，但是你会发现你无法创建文件或者目录，而提示却是空间不够，
这个时候你可以用 dumpe2fs 来看看你还有多少 free inode。当然仅仅为了查看 inode 节点的信息，不仅仅只有 dumpe2fs 能够
做得到。

`df -i` 命令也可以查看到 inode 节点的使用信息。

而适当的改变 block size 可以在同样空间里获得更多的 inode，这个我们在说 mkfs 的时候再讨论。

superblock 无疑是整个文件系统的核心，类似通往文件系统存储空间的钥匙，没有了钥匙，你既无法继续存储数据，也无法取出原来的数据。

正是因为他是如此的重要，所以文件系统在一创建的就做了若干备份。而且空间越大，备份的数目越多，以我的根文件系统为例，大小是` 5.3G`.

其中超级块一共 8 个。一旦文件系统的超级块损坏，我们立刻可以使用 `e2fsck -b xxxx` 的方式启用备用的超级块。

这里的 xxx 就是你的备份超级块的位置，上面的例子中，第一个超级块备份位置是 8193，因此你可以使用

`e2fsck -b 8193 /dev/sda1`

来恢复超级块。

除了这些重要信息外，对 dumpe2fs 的深入分析，甚至可以将一个一经无法通过恢复超级块来访问的文件系统的数据导出来。

## /sbin/ 下 e 开头的命令

/sbin/ 下 e 开头的命令比较多，列举如下：

```
e2fsck ela_remove elvtune evlfacility evlogmgr evSubagent
e2image ela_remove_all ether-wake evlgentmpls evlogrmtd
e2label ela_show ethtool evlnotify evlsend
ela_add ela_show_all evlactiond evlnotifyd evltc
ela_get_atts ela_sig_send evlconfig evlogd evlview
```

### e2fsck

检查和修复 ext2/ext3 文件系统，这似乎是目前 Linux 下唯一一个对文件系统做修复的工具。

他同时也是一个双刃剑。在实际的文件系统修复经验中，我遇到了两难的问题。根文件系统损坏了，需要修复，系统会提示你要用 e2fsck 来修复。
当然你可以按照他的提示去做，但是很可能，你会发现，等你修复完了，文件系统也能正常工作了，但是往往最重要的文件被修复得不加了。
这不是开玩笑，而是有大量的案例。如果在修复和损坏之间取得一种平衡需要根据具体的情况来定夺。

e2fsck 接受的参数比较多，列举如下：

- `-a`: 不询问使用者意见，便自动修复文件系统
- `-b`: 指定 superblock，而不使用预设的 superblock
- `-B`: 指定区块的大小，单位为字节
- `-c`: 一并执行 badblocks，以标示损坏的区块
- `-C`: 将检查过程的信息完整记录在 file descriptor 中，使得整个检查过程都能完整监控
- `-d`: 显示排错信息
- `-f`: 即使文件系统没有错误迹象，仍强制地检查正确性
- `-F`: 执行前先清除设备的缓冲区
- `-l`: 将文件中指定的区块加到损坏区块列表
- `-L`: 先清除损坏区块列表，再将文件中指定的区块加到损坏区块列表。因此损坏区块列表的区块跟文件中指定的区块是一样的
- `-n`: 以只读模式开启文件系统，并采取非互动方式执行，所有的问题对话均设置以"no"回答
- `-p`: 不询问使用者意见，便自动修复文件系统
- `-r`: 此参数只为了兼容性而存在，并无实际作用
- `-s`: 如果文件系统的字节顺序不适当，就交换字节顺序，否则不做任何动作
- `-S`: 不管文件系统的字节顺序，一律交换字节顺序
- `-t`: 显示时间信息
- `-v`: 执行时显示详细的信息
- `-V`: 显示版本信息
- `-y`: 采取非互动方式执行，所有的问题均设置以"yes"回答。

e2fsck 执行后的传回值及代表意义如下：

- 0: 没有任何错误发生。
- 1: 文件系统发生错误，并且已经修正。
- 2: 文件系统发生错误，并且已经修正。
- 4: 文件系统发生错误，但没有修正。
- 8: 运作时发生错误。
- 16: 使用的语法发生错误。
- 128: 共享的函数库发生错误。

### e2image

保存关键的 ext2/ext3 文件系统到文件。 e2image 可以用来创建 ext2 和 ext3 文件系统的镜像。  
e2image 只解释需要被镜像的文件系统，而不是保存原始 bit。  
e2image 可以创建 `raw` 和 `nomal` 镜像，这两种方法都可以节约空间。

因此，用 e2image 创建的镜像同硬盘上的文件系统有不同的 hash，从这点上看，他是和 dd 命令不同的。

但是如何将 e2image 创建的备份恢复出来呢？这点在 e2image 的 help 和 man 手册中都没有提到

下面是一个实际的例子，分部创建了 `raw` 和 `normal` 镜像

```shell
# e2image /dev/sda1 /tmp/e2image.data
e2image 1.35 (28-Feb-2004)

# ls -lh /tmp/e2image.data
-rw——- 1 root root 625K 1月 31 21:18 /tmp/e2image.data

# file /tmp/e2image.data
/tmp/e2image.data: Linux rev 1.0 ext3 filesystem data

# mount /tmp/e2image.data /misc -oloop
mount: wrong fs type, bad option, bad superblock on /dev/loop0,
or too many mounted file systems
(could this be the IDE device where you in fact use
ide-scsi so that sr0 or sda or so is needed?)

# mount -r /tmp/e2image.data /tmp/e2image_raw_data
mount: mount point /tmp/e2image_raw_data does not exist

# e2image -r /dev/sda1 /tmp/e2image_raw_data
e2image 1.35 (28-Feb-2004)

# ls -lh /tmp/e2image_raw_data
-rw——- 1 root root 20M 1月 31 21:20 /tmp/e2image_raw_data

# file /tmp/e2image_raw_data
/tmp/e2image_raw_data: Linux rev 1.0 ext3 filesystem data

# mount /tmp/e2image_raw_data /misc -oloop

# df -h
Filesystem 			  容量 已用 可用 已用% 挂载点
/tmp/e2image_raw_data 19M 1.2M 17M 7% /misc

# mount /dev/sda1 /misc

# df -h
Filesystem 容量 已用 可用 已用% 挂载点
/dev/sda1 19M 1.2M 17M 7% /misc
```

### e2label

显示或者修改 ext2/ext3 文件系统的标签。没有注意过从哪个版本开始，`/etc/fstab` 文件里的挂载设备名不再是实际的设备名称了，取而代之的是其标签，这增加了其灵活性，
特别是当删除并不是最后一个去分区时，在此分区前的设备号都会提前一位，如果 `/etc/fstab` 写入的是实际的设备名称，那显然这种情况下，
系统也许找不到需要的设备。而采用标签的话，只要标签不重复，即使设备有 sda 变成了 sdb，Linux 系统依然能正常启动。

这个命令非常简单。看下面的实例就明白了

```shell
# e2label /dev/sda1

# e2label /dev/sda1 newlabel

# e2label /dev/sda1
newlabel

```

### elvtune

I/O 电梯调试器，允许你调试每一个块设备的 I／O 电梯算法，但是还没有实现一路和二路电梯算法。

同时对于 LVM 而言，调试器仅仅在物理卷(PV)上有效果，对逻辑卷（LV）没有用。

不过这个命令要求的设备参数是/dev/blkdevX (X 表示数字），我尝试传递 /dev/sda1,/dev/hda1，均报告无效的参数。

所以没有觉得这个命令到底能给我们带来什么，特别是在 man 手册的历史一项中这样写道：

> Ioctls for tuning elevator behaviour were added in Linux 2.3.99-pre1.

感觉应该是一个临时工具，但是他属于 util-linux 工具包，不解。

### ether-wake

这个命令用来产生和发送一个 Wake-On-LAN(WOL)数据包(MagicPacker)包，用来重启软关机的机器。这个玩意没有办法测试，也不知道实际环境中是否用得多。

### ethtool

显示或修改以太网卡的设置，这里的设置真的是硬件方面的，比如强制为半双工等。

最开始我用这个命令是有一个用户说他的网卡明明是全双工的，为什么系统里看到的是半双工呀，有什么办法改吗？ifconfig 命令看了半天后没有找到好的设置方式，
后来找到了这个 ethtool，解决了问题。其实 mii-tool 也能做到这点，但是有些网卡驱动目前还与支持 mii-tool 工具。

ethtool 的命令比较复杂，参数也多，我们看几个例子，然后列出 man 手册。

**打印当前网卡的配置信息**

```shell
# ethtool eth0
Settings for eth0:
Supported ports: [ TP MII ]
Supported link modes: 10baseT/Half 10baseT/Full
100baseT/Half 100baseT/Full
Supports auto-negotiation: Yes
Advertised link modes: 10baseT/Half 10baseT/Full
100baseT/Half 100baseT/Full
Advertised auto-negotiation: No 注：自动协商关闭
Speed: 100Mb/s // 100Mb
Duplex: Full //全双工
Port: MII /支持mii模式
PHYAD: 32
Transceiver: internal
Auto-negotiation: off
Supports Wake-on: pumbg
Wake-on: d
Current message level: 0×00000007 (7)
Link detected: yes
```

**将网卡修改成 10Mbps，半双工**

```shell
# ethtool -s eth1 speed 10 duplex half
# ethtool eth1
Settings for eth1:
Supported ports: [ TP MII ]
Supported link modes: 10baseT/Half 10baseT/Full
100baseT/Half 100baseT/Full
Supports auto-negotiation: Yes
Advertised link modes: 10baseT/Half 10baseT/Full
100baseT/Half 100baseT/Full
Advertised auto-negotiation: No
Speed: 10Mb/s # 10Mbps
Duplex: Half #已经修改为半双工了
Port: MII
PHYAD: 32
Transceiver: internal
Auto-negotiation: off
Supports Wake-on: pumbg
Wake-on: d
Current message level: 0×00000007 (7)
Link detected: no
```

**将网卡修改为 100M，全双工**

```shell
# ethtool -s eth1 speed 100 duplex full
# ethtool eth1
Settings for eth1:
Supported ports: [ TP MII ]
Supported link modes: 10baseT/Half 10baseT/Full
100baseT/Half 100baseT/Full
Supports auto-negotiation: Yes
Advertised link modes: 10baseT/Half 10baseT/Full
100baseT/Half 100baseT/Full
Advertised auto-negotiation: No
Speed: 100Mb/s #
Duplex: Full #
Port: MII
PHYAD: 32
Transceiver: internal
Auto-negotiation: off
Supports Wake-on: pumbg
Wake-on: d
Current message level: 0×00000007 (7)
Link detected: no
```

我们再看看 man 手册

```
 ethtool ethX #显示网卡的
 ethtool -h #打印帮助
 ethtool -a ethX #查询指定网卡的暂停(pause)参数信息
 ethtool -A ethX [autoneg on|off] [rx on|off] [tx on|off] #设置暂停参数
 ethtool -c ethX #查询指定网卡的联合(coalesc)信息

 ethtool -C ethX [adaptive-rx on|off] [adaptive-tx on|off] [rx-usecs N]
 [rx-frames N] [rx-usecs-irq N] [rx-frames-irq N] [tx-usecs N] [tx-
 frames N] [tx-usecs-irq N] [tx-frames-irq N] [stats-block-usecs N]
 [pkt-rate-low N] [rx-usecs-low N] [rx-frames-low N] [tx-usecs-low N]
 [tx-frames-low N] [pkt-rate-high N] [rx-usecs-high N] [rx-frames-high
 N] [tx-usecs-high N] [tx-frames-high N] [sample-interval N] #设置指定网卡的联合信息

 ethtool -g ethX #查询网卡的RX/TX参数信息
 ethtool -G ethX [rx N] [rx-mini N] [rx-jumbo N] [tx N] #设置网卡的RX/TX参数
 ethtool -i ethX #查询与网卡关联的驱动信息
 ethtool -d ethX #重新获取和打印一个网卡的注册dump信息
 ethtool -e ethX [raw on|off] [offset N] [length N] #重新获取和打印网卡的EEPROMdump信息
 ethtool -E ethX [magic N] [offset N] [value N] #修改网卡的EEPROM字节
 ethtool -k ethX #查询网卡的卸载(offload)信息
 ethtool -K ethX [rx on|off] [tx on|off] [sg on|off] [tso on|off] #修改网卡的卸载参数
 ethtool -r ethX #重启网卡的自动协商(auto-negotiation)功能，如果这个功能设置位可用的话
 ethtool -t ethX [offline|online] #执行网卡自检程序
 ethtool -s ethX [speed 10|100|1000] [duplex half|full]
 [port tp|aui|bnc|mii] [autoneg on|off] [phyad N] [xcvr internal|exter-
 nal] [wol p|u|m|b|a|g|s|d...] [sopass xx:yy:zz:aa:bb:cc] [msglvl N] #可以修改的参数值，用-s来标志
```

这些设置仅仅在当前生效，一旦重启网络或者重启系统，ethtool 的设置将会失效。我们可以把 ethtool 的设置命令加入到 ifcfg-ethX 配置文件中。
比如，你希望把第一块网卡设置为全双工，自适应和 100M 的方式，那么你可以在你的 `/etc/sysconfig/network-scripts/ifcfg-eth0` 文件中加入下面一行：

```
ETHTOOL_OPTS="speed 100 duplex full autoneg off"
```

## /sbin 下 f 开头的命令如下

> fdisk fixfiles fsck.cramfs fsck.ext3 fsck.ocfs2 fsck.vfat fuser findfs  
> fsck fsck.ext2 fsck.msdos fsck.reiserfs fsck.xfs fxload

其中` fsck.reiserfs` 是 `reiserfsck` 命令的软连接，f 开头的命令中 fsck 命令套件占了绝大部分。

### findfs

根据标签(label)或者 UUID 号来查询文件系统。他会搜索整个磁盘，看是否有匹配的标签或者 UUID 没有，如果有，则打印到标注输出上。他也是[e2fsprogs](http://e2fsprogs.sf.net)项目的一部分。

一个简单的例子：

```shell
# findfs –help
Usage: findfs LABEL=|UUID=

# findfs LABEL=/
/dev/hda5

# findfs LABEL=/home
findfs: Unable to resolve 'LABEL=/home'
```

我没有搞清楚这个命令到底有什么用或者应该在什么情况下使用？因为知道 LABEL 或者 UUID 并不像平常知道某一个文件那样容易记住。所以他和 find 的使用范围相比，就小了很多了，难道他有意想不到的用途？

### fixfiles

修复文件安全上下文(security contexts)。这是一个 shell 脚本，主要是用来校正文件系统上的安全上下文数据库（扩展属性）。
仔细看了帮助，也作了一些测试，没有明白他要表达什么，刚开始以为是他会恢复我原来修改过某些配置文件到初始状态，当经过测试，似乎不是这回事。
所属于 policycoreutils RPM 包，这是 SELinux 的一部分。

### fuser

使用文件或者套节字来表示识别进程。我常用的他的两个功能：查看我需要的进程和我要杀死我查到的进程。

比如当你想 umount 光驱的时候，结果系统提示你设备正在使用或者正忙，可是你又找不到到底谁使用了他。这个时候 fuser 可派上用场了。

```shell
# eject
umount: /media/cdrom: device is busy
umount: /media/cdrom: device is busy
eject: unmount of '/media/cdrom' failed

# fuser /mnt/cdrom
/mnt/cdrom: 4561c 5382c

# ps -ef |egrep '(4561|5382)' |grep -v grep

root 4561 4227 0 20:13 pts/1 00:00:00 bash
root 5382 4561 0 21:42 pts/1 00:00:00 vim Autorun.inf
```

示例中，我想弹出光驱，系统告诉我设备忙着，于是采用 fuser 命令，参数是你文件或 scoket，fuser 将查出那些使用了他。

`4561c,5382c` 表示目前用两个进程在占用着 `/mnt/cdrom`，分别是 4561,5382,进程 ID 后的字母表示占用资源的方式，有下面几种表示：

- c 当前路径(current directory.)我的理解是表示这个资源的占用是以文件目录方式，也就是进进入了需要释放的资源的路径，这是最常用的资源占用方式。
- e 正在运行可执行文件（executable being run.），比如运行了光盘上的某个程序
- f 打开文件（ open file），缺省模式下 f 忽略。
- m mmap 文件或者共享库( mmap’ed file or shared library)
- 所以上面的例子中，虽然是开打了光盘上的 Autorun.inf 文件，但是给出的标识是 c，而不是 f。
  root 目录（root directory）.没有明白什么意思，难道是说进入了/root 这个特定目录？  
  .这应该是说某个进程使用了你要释放的资源的某个共享文件。

在查找的同时，你还可定指定一些参数，比如

- `-k`: 杀死这些正在访问这些文件的进程。除非使用-signal 修改信号，否则将发送 SIGKILL 信号。
- `-i`: 交互模式
- `-l`: 列出所有已知的信号名称。
- `-n`: 空间，选择不同的名字空间，可是 file,udp,tcp。默认是 file，也就是文件。
- `-signal`: 指定发送的信号，而不是缺省的 SIGKILL
- `-4`: 仅查询 IPV4 套接字
- `-6`: 仅查询 IPV6 套接字

再看下面的例子

```shell
# fuser -l
HUP INT QUIT ILL TRAP ABRT IOT BUS FPE KILL USR1 SEGV USR2 PIPE ALRM TERM
STKFLT CHLD CONT STOP TSTP TTIN TTOU URG XCPU XFSZ VTALRM PROF WINCH IO PWR SYS
UNUSED
```

现在我们试试 fuser -k 的威力：

```shell
# fuser -k /mnt/cdrom
/mnt/cdrom: 4561c 5382c

kill 5382: 没有那个进程
No automatic removal. Please use umount /media/cdrom

# eject
```

套节字方式的使用：

```shell
# fuser -4 -n tcp 3306
here: 3306
3306/tcp: 5595

# ps -ef |grep 5595 |grep -v grep

mysql 5595 5563 0 22:24 pts/0 00:00:00 /usr/libexec/mysqld –defaults-file=/etc/my.cnf –basedir=/usr –datadir=/var/lib/mysql –user=mysql –pid-file=/var/run/mysqld/mysqld.pid –skip-locking –socket=/var/lib/mysql/mysql.sock


# fuser -4 -n tcp 80
here: 80
80/tcp: 5685 5688 5689 5690 5691 5692 5693 5694 5695
```

### fxload

为 EZ-USB 设别下载 firmware。

## /sbin 下 g 开头的命令：

> generate-modprobe.conf getkey grub grub-install grub-terminfo genhostid
> getmd grubby grub-md5-crypt

### generate-modprobe.conf

将原来的 module.conf 文件迁移到 modprobe.conf 文件。这是 2.4 内核到 2.6 内核后驱动模块配置的一个改变。

### getkey

在指定的时间内等待按键，默认时间单位是秒。比如：

```shell
# getkey -c 5 -m "Please input a number within 5s[%d]:"
Please input a number within 5s[5]
```

后面的`[]`内的时间是变化的。

### grub

grub, grub-install ,grub-terminfo, grub-md5-crypt,grubby 这这些命令都是和 grub 有关的。

### genhostid

随机生成一个 hostid 号，如果 `/etc/hostid` 不存在的话，他和我们通常使用的 hostid(1)有些微妙的关系。

如果 `/etc/hostid` 文件不存在，则 hostid 命令打印出当前机器的标志号，只要你的机器硬件设备没有更换，这个 ID 号就不会改变，很多情况下这个号码和你的网卡有一定的关系。  

于是在 Linux 下，很多软件的授权号码就于 hostid 有关系，因为不同的机器 hostid 不同，这样能保证只有授权的机器才能安装对应的软件。  

但是如果 `/etc/hostid` 文件存在, hostid 命令则只是打印出该号码，而 `/etc/hostid` 的生成就是依赖 genhostid 命令。  

但是我刚才说了，genhostid 是随机产生一个 hostid 号。
因此如果你希望 hostid 命令每次得到不同的结果，那么你就应该每次都重新生成 /etc/hostid 文件。不过要注意的是 hostid 命令生成的 hostid 号只有 6 位，而 genhostid 产生的是 8 位。

```shell
# hostid
7f0100

# genhostid
# hostid
804937a4

# rm -f /etc/hostid
# hostid
7f0100

# genhostid
# hostid
106a8419

# hexdump /etc/hostid
0000000 8419 106a
0000004
```

从 hexdump 命令的结果来看，`/etc/hostid` 文件结构似乎很简单，我们尝试获得一个我们需要的“随机”hostid 号试试。

```c
// myhostid.c
#include
#include
int main()
{
	int fd1;
	int a=0×12345678; //my hostid
	fd1=open(“hostid”,O_CREAT|O_WRONLY,0220);
	if (!fd1)
	{
		perror(“error open “);
		exit(65);
	}
	write(fd1,&a,sizeof(a));
	close(fd1);
	return 0;
}
```

```shell
# gcc myhostid.c -o myhostid
# ./myhostid

# #cp -f hostid /etc/hostid
# hostid
d2bdd8ed

# cp -f hostid /etc/hostid
# hostid
12345678
```

### getmd

分析`/etc/raidtab`文件，它是 scsirastools 包中一个命令，这个包的主要功能就是增加一些管理 scsi 设备的命令，有下面一些：

* `sgdskfl`:   to load disk firmware to disks under Linux  
* `sgmode`:   to get and set SCSI mode pages  
* `sgdefects`:   to read the primary and grown defect lists  
* `sgdiag`:   to perform format & other test functions  
* `sgraidmon`:   to monitor software RAID disks for hot-insertion/removal  
* `getmd`:   to parse /etc/raidtab for devices (used in mdmon only)  
* `mdadm`:   to administer the md raid configuration  
* `alarms`:   to set disk fault LEDs on mBMC platforms

## /sbin 下 h 开头的命令

### halt 
停止系统，他和 reboot，poweroff 等命令属于同一个功能集。

halt 命令不不接参数，仅仅有几个选项来决定如何停止系统

* `-n`: 停止系统前不做同步(sync)，适合快速关机和那些仅仅只有只读访问的系统。  
* `-w`: 仅在 wtmp 文件中写一条 reboot 记录，然后退出，他的用处是什么？  
* `-d`: 不写 reboot 记录到 wtmp 文件中  
* `-f`: 强制停机/重启，但不调用 shutdown.shutdown 是一种安全的关闭系统的方式。  
* `-p`: 如果支持，系统关闭后掉电。否则只是停止系统.


### hdparm

获取和设置硬盘参数，当我一次看到 hdparm 命令和其说明时，我以为我发现了一个秘笈，因为说明中提到，他可以设置一些关键的参数，让你的硬盘提速。
同时在网上也找到了不少有关这个命令的介绍，大多数都说可以显著提高硬盘响应速度，还有很多人的标题干脆是“瞬间提高你的 Linux 的速度”。
其实大部分介绍中，主要是修改两个参数，一个是设置 DMA 为 enable，另外一个是设置 I/0 为 32 位。  
我测试过这两个参数，对硬盘的响应速度确实有很大的影响，特别是 DMA 的设置，有相差好几倍的效果。  

然后实际上，后来的 Linux 分发版本，对于硬盘的优化早在系统启动中就已经做好了。  

而我知道的是，仅仅只有 2.2 的内核默认是不打开 DMA 模式的，因此在 2.2 的内核，利用`hdparm -c 1 /dev/hdx`的方式来提高硬盘响应速度是有效果的，但是 2.2 的内核毕竟是古董了。  

另外 hdparm 的 man 提到 hdparm 仅仅只是设置 ATA/IDE 硬盘参数，而对 SCSI 硬盘是没有用的，当然也对后来出现的 SATA,SAS 硬盘无效了。 
我没有测试过 SCSI 硬盘，也没有测试过 SATA,SAS 硬盘，所以现在还不能给出一个确切的答案。

我们看看后来的 linux 版本是如何在启动是针对不同硬盘做的优化的，下面的脚本片段来自 `/etc/rc.d/rc.sysinit` 文件

```shell
# Turn on harddisk optimization
# There is only one file /etc/sysconfig/harddisks for all disks
# after installing the hdparm-RPM. If you need different hdparm parameters
# for each of your disks, copy /etc/sysconfig/harddisks to
# /etc/sysconfig/harddiskhda (hdb, hdc…) and modify it.
# Each disk which has no special parameters will use the defaults.
# Each non-disk which has no special parameters will be ignored.
#

disk[0]=s;
disk[1]=hda; disk[2]=hdb; disk[3]=hdc; disk[4]=hdd;
disk[5]=hde; disk[6]=hdf; disk[7]=hdg; disk[8]=hdh;
disk[9]=hdi; disk[10]=hdj; disk[11]=hdk; disk[12]=hdl;
disk[13]=hdm; disk[14]=hdn; disk[15]=hdo; disk[16]=hdp;
disk[17]=hdq; disk[18]=hdr; disk[19]=hds; disk[20]=hdt;

if [ -x /sbin/hdparm ]; then
	for device in 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20; do
		unset MULTIPLE_IO USE_DMA EIDE_32BIT LOOKAHEAD EXTRA_PARAMS
		if [ -f /etc/sysconfig/harddisk${disk[$device]} ]; then
			. /etc/sysconfig/harddisk${disk[$device]}
			HDFLAGS[$device]=
			if [ -n “$MULTIPLE_IO” ]; then
			HDFLAGS[$device]=”-q -m$MULTIPLE_IO”
			fi
			if [ -n “$USE_DMA” ]; then
			HDFLAGS[$device]=”${HDFLAGS[$device]} -q -d$USE_DMA”
			fi
			if [ -n “$EIDE_32BIT” ]; then
			HDFLAGS[$device]=”${HDFLAGS[$device]} -q -c$EIDE_32BIT”
			fi
			if [ -n “$LOOKAHEAD” ]; then
			HDFLAGS[$device]=”${HDFLAGS[$device]} -q -A$LOOKAHEAD”
			fi
			if [ -n “$EXTRA_PARAMS” ]; then
			HDFLAGS[$device]=”${HDFLAGS[$device]} $EXTRA_PARAMS”
			fi
		else
			HDFLAGS[$device]=”${HDFLAGS[0]}”
		fi

		if [ -e “/proc/ide/${disk[$device]}/media” ]; then
			hdmedia=`cat /proc/ide/${disk[$device]}/media`
			if [ “$hdmedia” = “disk” -o -f “/etc/sysconfig/harddisk${disk[$device]}” ]; then
				if [ -n “${HDFLAGS[$device]}” ]; then
					action $”Setting hard drive parameters for ${disk[$device]}: ” \
						/sbin/hdparm ${HDFLAGS[$device]} /dev/${disk[$device]}
				fi
			fi
		fi
	done
fi
```

这段脚本是所有的 IDE 硬盘，根据 `/etc/sysconfig/harddisks` 文件的策略而设置硬盘的参数. 
从这里我们似乎可以看出，sdx 设备并不再设置之内，由此是不是可以肯定 hdparm 对 SCSI 设备不支持呢？
我们再看看 harddisks 文件

```ini
#
# WARNING !!!
#
# The kernel will autodetect the correct settings for most drives.
# Overriding these settings with hdparm can cause data corruption and/or
# data loss.
# Leave this file as it is unless you know exactly what you're doing !!
#
#
#
# These options are used to tune the hard drives -
# read the hdparm man page for more information

# Set this to 1 to enable DMA. This might cause some
# data corruption on certain chipset / hard drive
# combinations. This is used with the “-d” option

# USE_DMA=1

# Multiple sector I/O. a feature of most modern IDE hard drives,
# permitting the transfer of multiple sectors per I/O interrupt,
# rather than the usual one sector per interrupt. When this feature
# is enabled, it typically reduces operating system overhead for disk
# I/O by 30-50%. On many systems, it also provides increased data
# throughput of anywhere from 5% to 50%. Some drives, however (most
# notably the WD Caviar series), seem to run slower with multiple mode
# enabled. Under rare circumstances, such failures can result in
# massive filesystem corruption. USE WITH CAUTION AND BACKUP.
# This is the sector count for multiple sector I/O – the “-m” option
#
# MULTIPLE_IO=16

# (E)IDE 32-bit I/O support (to interface card)
#
# EIDE_32BIT=3

# Enable drive read-lookahead
#
# LOOKAHEAD=1

# Add extra parameters here if wanted
# On reasonably new hardware, you may want to try -X66, -X67 or -X68
# Other flags you might want to experiment with are -u1, -a and -m
# See the hdparm manpage (man hdparm) for details and more options.
#
EXTRA_PARAMS=
```

从这个文件的注释我们能够看出，它将影响硬盘效果最明显的参数单独列了出来，而将其他参数放到了 EXTRA_PARAMS 里。  

因此对于 hdparm 命令的众多参数而言，我们需要去了解的不太多。  

### hotplug

Linux 下热插拔支持脚本。  

### hwclock

查询和设置硬件时钟(RTC).系统启动时，需要从硬件时钟中读出时间来设置系统系统，这个过程在`/etc/rc.d/rc.sysinit` 脚本中采用`hwclock $CLOCKFLAGS`的方式来实现。

这里的参数 `$CLOCKFLAGS` 分为两部分，一部分是 `--hctosys`,表示同步的方式是硬件时钟到系统时钟。另外一部分参数是时间设置的标准，是标准时间（utc），还是本地时间(localtim). 

而关闭系统时，则相反，似乎用 `-systohc` 的参数来将系统时钟同步到硬件时钟。

## /sbin/下 l 开头的命令

不多，列举如下：

> ldconfig lilo logsave loopctrl losetup lsmod lspci lsusb lvm lvm.static

其中 lvm 是 lvm.static 的软连接。

### ldconfig

配置动态连接库，以便程序在编译和运行中动态连接这些程序库。与此相关的配置文件便是 `/etc/ld.so.conf`，
后来的 ldconfig 的配置文件采取了现在流行的读取目录下的配置文件。所以 `/etc/ld.so.conf.d`目录便是后来增加的。

我仅仅使用 ldconfig 的一个参数-f，就是指定配置文件，这样的好处是，当我需要增加某一个目录的动态库时，我不要全部重建动态库缓存，命令程序运行时间短了很多。

### lilo

grub 之前的引导程序，不知道现在还有多少发行版本使用他,留着他的一个好处是，当我丢失引导程序，而 grub 又安装不上时，lilo 这个时候就可以出马了，而 lilo 每次都没有让我失望，所以我对他的印象不错。

### logsave

日志记录程序，参数方式如下：

```shell
logsave [ -asv ] logfile cmd_prog [ ... ]
```

* `-a`: 是采用附加的模式写入日志文件 logfile  
* `-s`: 写日志到用户终端  
* `-v`: 就是冗余输出

给我的感觉他的作用类似重定向，比如

```shell
cmd_prog >logfile 2>&1
```

### loopctrl

配置 isdnloop ISDN 驱动，这应该算是一个用户遗留产品。

### lsmod

显示目前加载的内核模块状态

### lspci

列出所有的 PCI 设备，这个命令我用得比较少


### lsusb

列出所有的 usb 驱动

### lvm

lvm,lvm.static 逻辑卷管理程序程序, 他其实包括了一系列的子命令，比如 

* pvscan
* pvcreate
* pvdisplay
* vgscan
* vgcreate
* vgchange
* vgdisplay
* lvscan
* lvcreate
* lvdisplay 

等，这些子命令也可以单独使用，从一个初始的硬盘开始到创建一个可以使用的文件系统并挂载的简单步骤是:(我这里用 /dev/ram0 来替代硬盘)

```shell
# pvcreate /dev/ram0
Physical volume “/dev/ram0″ successfully created

# vgcreate vgname /dev/ram0
Volume group “vgname” successfully created

# vgchange -a y vgname
0 logical volume(s) in volume group “vgname” now active

# lvcreate -L20M -n lvname vgname
Logical volume “lvname” created

# mkfs.ext3 /dev/mapper/vgname-lvname
mke2fs 1.39 (29-May-2006)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
5136 inodes, 20480 blocks
1024 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=20971520
3 block groups
8192 blocks per group, 8192 fragments per group
1712 inodes per group
Superblock backups stored on blocks:
8193

Writing inode tables: done
Creating journal (1024 blocks): done
Writing superblocks and filesystem accounting information: done

This filesystem will be automatically checked every 39 mounts or
180 days, whichever comes first. Use tune2fs -c or -i to override.

# mount /dev/mapper/vgname-lvname /misc

# df -h
文件系统 容量 已用 可用 已用% 挂载点
/dev/hda5 6.7G 2.2G 4.2G 35% /
/dev/hda1 99M 23M 71M 25% /boot
/dev/shm 248M 0 248M 0% /dev/shm
/dev/hda3 6.8G 5.3G 1.2G 83% /dc50
/dev/mapper/vgdata-data
20G 831M 18G 5% /data
/dev/mapper/vgname-lvname
20M 1.2M 18M 7% /misc
```

## /sbin 目录下 m 开头的命令

可不少

```shell
mdadm          mii-tool    mkfs.cramfs    mkfs.xfs    mounted.ocfs2  mpath_get_name          multipath.static
mdadm.static   mingetty    mkfs.ext2      mkinitrd    mount.ncp      mpath_prio_alua
mdassemble     minilogd    mkfs.ext3      mkreiserfs  mount.ncpfs    mpath_prio_emc
mdevt          mkbootdisk  mkfs.msdos     mkswap      mount.ocfs2    mpath_prio_hds_modular
mdmpd          mkdosfs     mkfs.ocfs2     mkzonedb    mount.smb      mpath_prio_netapp
mgetty         mke2fs      mkfs.reiserfs  modinfo     mount.smbfs    multipath
microcode_ctl  mkfs        mkfs.vfat      modprobe    mpath_ctl      multipathd
```

mkfs 系列命令占了很大一部分。 

### mdadm

mdadm, mdadm.static 这是和 RAID 相关的程序，他用来管理 Linux 下的软 RAID。

当前 Linux 提供的 RAID 级别有 

- LINEAR md 设备
- RAID0 (striping)
- RAID1 (mirroring)
- RAID4
- RAID5
- RAID6
- MULTIPATH
- FAULTY


这里说的 multipath 其实并不是软 RAID 机制中的一部分，但是他会去调用。

### mdmpd

监控 MD multipath 设备的 daemon 程序。  

### mii-tool

查看，管理独立介质接口状态，这是一个过时的程序了，目前代替他功能的程序是 eth-tool。  

### mkbootdisk

制作启动磁盘，不知道现在还有多少人使用他，现在看到的服务器似乎都没有软驱了。  

### mkfs.\*：

这是制作文件系统的程序集，如果什么都不考虑，仅仅是制作文件系统的话，那直接 mkfs 了。但是如果你要从存储数据的性质，读写速度，存放文件个数等方面着想，那就不得不考虑 mkfs 命令集的一些参数了。

### mkinitrd

重做镜像文件，这个命令在更新了一些核心驱动模块的时候，用得着。
比如更新了 SCSI 卡的驱动，增加驱动模块的一些参数等，一般来说，如果修改了`/etc/modprobe.conf` 中块设备文件的配置参数，就应该重新制作镜像文件，

方法倒是很简单 

```shell
mkinitrd /boot/newinitrd.img `uname -r`
```

### modinfo

显示内核模块的一些信息，包括作者，描述，授权方式，核心参数等，其中有的还包括了模块依赖性的描述，比如:

```shell
# modinfo e100.ko
filename:       /lib/modules/2.6.9-42.7AX/kernel/drivers/net/e100.ko
description:    Intel(R) PRO/100 Network Driver
author:         Copyright(c) 1999-2005 Intel Corporation
license:        GPL
version:        3.5.10-k2-NAPI 6481838CE42D9570A7D35AF
parm:           debug:Debug level (0=none,...,16=all)
vermagic:       2.6.9-42.7AX 686 REGPARM 4KSTACKS gcc-3.4
depends:        mii
alias:          pci:v00008086d00001029sv*sd*bc02sc00i*
alias:          pci:v00008086d00001030sv*sd*bc02sc00i*
alias:          pci:v00008086d00001031sv*sd*bc02sc00i*
alias:          pci:v00008086d00001032sv*sd*bc02sc00i*
alias:          pci:v00008086d00001033sv*sd*bc02sc00i*
alias:          pci:v00008086d00001034sv*sd*bc02sc00i*
.........................................
```

从这个命令的输出我们可以看出 `e100.ko` 这个模块目前的版本是 3.5.10，他依赖模块 `mii.ko`，同时，可以传递一个参数 debug，用于调试。

### mount.\*

文件系统挂载命令，这些命令的前端程序就是 mount,类似 mkfs 一样。  

这个命令恐怕是 linux 下最常用的命令之一了。所以大家都比较熟悉，我这里仅仅记录几个平常不怎么常用的几个参数

**重新挂载为只读(可写)**

```shell
# mount hd /misc -oloop
# touch /misc/one
# ls /misc
lost+found  one
# mount -o remount,ro hd /misc -oloop
# touch /misc/two
touch: cannot touch `/misc/two': Read-only file system
```

**猜测某个文件系统的类型**

```shell
# mount --guess-fs /dev/hda3
ext3
```

## /sbin 下 n 开头的命令

不多，而且基本上不常用

> nameif nash netplugd netreport new-kernel-pkg nologin

### nameif

基于 MAC 地址命名网络接口，他需要的配置文件是 `/etc/mactab`。

### nash

另外一种 shell，主要用在引导镜像中的脚本中，比如`initrd.img`。

### netplugd

网络热插拔管理后台程序。

### new-kernel-pkg

安装新内核，他能完成的功能有：  

1. 创建，删除一个 initrd  
2. 运行 depmod 或者删除 depmod 生成的文件  
3. 从 grub/lilo 配置文件中增加或者删除一个内核镜像（通过 grubby）。

### nolog

想拒绝一个帐号登录，就可以使用他了，man 手册中描述的是优雅的拒绝用户登录。


跳过 o 开头的命令是因为 o 开头只有三个命令，而且都是和 Oracle 的 ocfs 有关，所以这里就不研究了。


## /sbin/p 开头的命令

p 开头的命令不少，近 20 个。

### pack_cis

与 PCMCIA 有关，编译 PCMCIA 卡信息结构。

### pam_cosonle_apply

在系统终端上，针对用户设置或者收回权限。他是 PAM 相关的工具。

如果`/var/run/console.lock`存在，则针对该文件的用户授权;如果没有，则根据缺省文件
`/etc/security/console.perms`的规则来授权。  

这个命令以及`pam_timestamp_check`,`pam_tally`都属于 PAM 程序

### pam_console_setowner

设置 console 的属主，他属于 UDEV 包.

### parted 

强大的磁盘管理工具，类似 fdisk 功能，但是个人觉得其功能比 fdisk 强大。
其中值得说的一点就是在已经占用的磁盘上进行操作，不需要重启生效，这点就比 fdisk 强。比如你的 hda 已经在使用了（你的根文件系统），这时你希望在 hda 上再创建一个分区，如果使用 fdisk 工具，保存的时候肯定会提示要你重启才生效，因为设备 busy。
但是 parted 却不会。
另外一点是 fdisk 程序只有在你输入 `w` 命令后，所有的命令才真实生效，而 parted 不是这样，执行后，立刻生效（也许我还没有找到 undo 的功能），所以这点要注意。

发现一个有意思的现象：addpart，delpart 的功能是删除分区和增加分区，那么很自然的应该属于 parted 程序的功能，因此按理，这两个命令连带 partprobe 和 parted 属于同一个包。
但事实是 addpart,delpart 属于 util-linux 包。不知道为什么要这么来归类？

Linux 下没有图形化的磁盘管理工具一直是普通用户抱怨的地方，因为不要期望所有人都来使用 fdisk，parted 这样的命令。不过现在好了，给予 parted 的[qtparted](http://qtparted.sf.net)就是一个 GUI 的程序。

### partx

partx 同样属于 parted 的包，而是 linux-util,虽然长像 parted 之类的。这个命令和 addpart，delpart 一样，没有 man 手册，没有 info 信息，没有 `--help`。

只好找 google，得到这个这么[一篇文章](http://www.ussg.iu.edu/hypermail/linux/kernel/0003.2/1887.html)。可以看看。

### pivot_root

改变根文件系统。他的基本用法是：`pivot_root new_root put_old`  

表示把当前处理的根文件系统移到`put_old`目录，然后把`new_root`作为当前的根文件系统。  

这个命令更多的时候用在系统启动时。我们知道现在的 Linux 启动时，都需要一个 initrd 的核心参数，这个 initrd 后面接着的压缩文件很多情况下就是一个文件系统。  

内核引导后，会首先把 initrd 的参数挂载，把他当成根文件系统，处理完必要的事务后，再把核心参数 root 置顶的设备挂载上来，然后使用`pivot_root`命令切换整个根文件系统，
接着执行/sbin/init 程序来继续引导系统。

LSB 的 linux 发行版本，引导时会把 initrd 镜像挂载到 `/initrd` 目录下。所以你会发现根目录下有一个 initrd 目录，但是他是空的，如果我删除他，系统引导的时候就会报错，我怎么知道的？因为我干过这样的傻事。

man 手册中举了几个例子，可以进一步解释`pivot_root`的作用。不过对于改变根文件系统，我们可能首先想到的是 chroot，chroot 似乎也有这个功能，那么他和`pivot_root`有什么差别呢？  
主要的差别是`pivot_root`有意识的切换一个完整的系统到新的根目录，然后可以移走之前需要依赖的老的系统，因此你可以 umount 掉之前的根文件系统，就当做之前没有发生过原来它是运行的操作系统一样。

而 chroot 则有意识的申请一个单独进程的全部生命周期，他和系统的其他部分一起在老的根目录下一起运行，当 chrooted 进程退出后，系统不会发生改变。

注：不建议大家测试这个命令，这时我测试后得出的结果。至少我做完后，我是靠重启系统来恢复原状的(真实原因是对这个命令还是了解得不太多。)

### portmap

DARPA(美国国防部高级研究计划局)端口到 RPC 程序号的映射。也就是提供把 RPC 程序号转换成 DARPA 协议端口的服务。更统属的称呼应该就是端口映射(谁关心到底从什么协议转什么协议呢？)。 

当一个 RPC 服务开始时，它告诉 portmap 它将侦听哪个端口号，以及它准备为什么 RPC 程序号提供服务。  
但一个客户端希望对指定的程序号做 RPC 调用时，他将首先链接服务器上的 portmap 程序(服务)来决定 RPC 包应该发往哪个端口。  
portmap 应该在 RPC 服务调用之前启动。  

典型的应用应该是 NFS 了。运行一个 NFS 服务当前需要启动的服务有：portmap，nfs 和 nfslock。不启动 portmap 而单独启动 nfs， fslock 服务提供 NFS 服务时，我遇到过两种错误：

1. 非常长的时间才能 mount 上  
2. 直接报给出的服务不可用。

### /sbin 下的 q 开头的命令

只有两个，quotacheck 和 quotaon，其中 quotaoff 是 quotaon 的软连接。

这是和磁盘配额有关的命令，详细情况可以参考 Linux 下的配额配置手册。

### /sbin 下 r 开头的命令

稍微多些，有 25 个，不过有 7 个是软连接。


### restore

从 dump 出来的备份恢复到文件系统中，我仅测试了一个例子，看上去一切正常，不过从这个例子。如果不考虑一些特别因素，那么 dump/restore 的使用方法是很简单的，下面是我测试的例子

```shell
#mount /dev/mapper/vgdata-lvtest /misc

#cp -a /boot/* /misc/

#umount /misc

#dump z9 -f /tmp/lvtest.dump /dev/mapper/vgdata-lvtest
DUMP: Date of this level dump: Mon May 21 10:34:31 2007
DUMP: Dumping /dev/mapper/vgdata-lvtest (an unlisted file system) to /tmp/lvtest.dump
DUMP: Label: none
DUMP: Writing 10 Kilobyte records
DUMP: Compressing output at compression level 9 (zlib)
DUMP: mapping (Pass I) [regular files]
DUMP: mapping (Pass II) [directories]
DUMP: estimated 40454 blocks.
DUMP: Volume 1 started with block 1 at: Mon May 21 10:34:33 2007
DUMP: dumping (Pass III) [directories]
DUMP: dumping (Pass IV) [regular files]
DUMP: Closing /tmp/lvtest.dump
DUMP: Volume 1 completed at: Mon May 21 10:34:44 2007
DUMP: Volume 1 took 0:00:11
DUMP: Volume 1 transfer rate: 3056 kB/s
DUMP: Volume 1 40530kB uncompressed, 33626kB compressed, 1.206:1
DUMP: 40530 blocks (39.58MB) on 1 volume(s)
DUMP: finished in 11 seconds, throughput 3684 kBytes/sec
DUMP: Date of this level dump: Mon May 21 10:34:31 2007
DUMP: Date this dump completed: Mon May 21 10:34:44 2007
DUMP: Average transfer rate: 3056 kB/s
DUMP: Wrote 40530kB uncompressed, 33626kB compressed, 1.206:1
DUMP: DUMP IS DONE

# ls -lh /tmp/lvtest.dump
-rw-r--r-- 1 root root 33M 05-21 10:34 /tmp/lvtest.dump

# mkfs.ext3 /dev/mapper/vgdata-lvtest
# mount /dev/mapper/vgdata-lvtest /misc
# cd /misc

# restore rf /tmp/lvtest.dump
Dump tape is compressed.
/dc50/sbin/restore: ./lost+found: File exists
[root@mlsx misc]# ls
boot.b initrd-2.6.18.3-52.img memtest86+-1.65 System.map-2.6.18.3-52smp
vmlinuz-2.6.18.3-52smp
chain.b initrd-2.6.18.3-52smp.img message System.map-2.6.20.6-2
vmlinuz-2.6.20.6-2
config-2.6.18.3-52 initrd-2.6.20.6-2.img module-info
System.map-2.6.20-ovz005.1 vmlinuz-2.6.20-ovz005.1
config-2.6.18.3-52smp initrd-2.6.20-ovz005.1.img newinitrd.img
System.map-2.6.9-42.7AX vmlinuz-2.6.9-42.7AX
config-2.6.20.6-2 initrd-2.6.9-42.7AX.img os2_d.b
vmlinux-2.6.20-ovz005.1
config-2.6.9-42.7AX initrd.img restoresymtable vmlinuz
grub lost+found System.map-2.6.18.3-52 vmlinuz-2.6.18.3-52
```

从上面的例子来看，似乎 dump/restore 的使用方式和 tar 差不多，但是没有 tar 那么具有亲和力，毕竟 tar 是更加通俗的备份工具。
但是从技术角度来说，dump/restore 和 tar 存在质的区别：dump/restore 是基于块设备的备份方式，而 tar 则是同时文件系统。

这样的话，理论上dump 的速度应该比 tar 的速度要块写。而且因为是从块设备备份，因此也更灵活，提供的参数也特别多。

当时说到备份恢复，在 Unix 和 Linux 系统中，有太多的工具了。

除了这两者外，还有一些不太常见的，比如 cpio，这个文件格式在 Oracle 官方提供的介质下载中能看到，还有一个pax，Portable Archive eXchange 的缩写。

有关备份的工具，我找到一篇文档，大家可以看看

> Unix 系統基本的備份與回復工具—dump 及 restorerunuser: 用指定的账号或者组来运行一个 shell，类似 su 程序，但是他没有密码提示。上面的解释是 man 手册中的翻译，后面的哪个密码提示不怎么好理解，为什么呢？我们知道 su 程序从 root 账号切换到其他账号是不需要密码的，而普通账号之前的切换或者从普通账号切换到 root 账号，你需要提供密码的。 因为能从普通账号切换到 root 账号，因此意味着 su 命令是具有 setuid 的，而 runuser 却没有。所以当在 root 账号下执行 runuser test 是没有问题的，立刻切换到账号 test 环境下，但是这个时候你是无法从 test 账号切换切换到任何账号的，他会给你权限不够的提示。当然即便提供输入密码，也没戏，因为 runuser 是一个普通命令，没有 setuid 位，那就无法从普通账号切换了。所以我不清楚了有 su 这样的命令后，为什么还需要 runuser 这样的命令，历史原因？向后兼容？Unix 有？r 开头的其他命令这里不说了，有好几个是于 reiser 文件系统相关的命令，自从 reiser 文件系统的作者背叛谋杀妻子罪成立后，reiser 文件系统的命运就不好说了，据说作者准备出售 reiser 文件系统的版权来打官司。唉，还是等等看吧。另外就是 rmmod 命令，这是内核驱动卸载命令，我记得很早前的 rmmod 是 insmod 命令的一个软连接，现在不是了，看来他们分家了。但是分家的原因未知。rfmond 是 RedFlag 系统特有的，是红旗系统日志收集后台程序，前台是 GUI 界面的。rpc 开头的命令不少，这些命令更多的应该是被其他 rpc 应用程序来调用，比如 smb，比如 nfs。runlevel 比较熟悉，参数也仅有一个，它是从/var/run/utmp 文件中猜着当前运行的级别和之前运行的级别。比如你是在 3 级别登录的，然后执行 startx，然后你运行 runlevel，那么他给出的答案是 5 3 表示现在是 5 级别，之前是 3 级别。其实从 3 级别执行 startx 来启动图形级别，我更愿意说是在 3 级别启动了一个应用程序，这是这个应用程序比较特别，是一个 X-windows，从命令角度来说，他应该和在 3 级别执行 ls 等命令没有什么差别。所以我倒不希望 startx 后，运行 runlevel 给出是答案是 5 3,我期望是 3.这样 startx 这个命令本身就没有什么特别性了。我这里来说，是比较好向 Linux 新手解释 Linux 的 X-windows。我理解的从 3 级别到 5 级别应该采用最原始，最传统的 init 5 命令。

### sfdisk 

有四个主要的功能：列出一个分区的大小（块为单位），一个设备的所有分区，检查一个设备的分区表，重新分区一个设备。

其实我在平时用的最多的是第二个--列出一个设备的分区表。有了 fdisk 命令，为什么还要有 sfdisk 命令呢？我第一次具体使用 sfdisk 命令，还是在 HP DL 系列机器上。
HP 机器使用了 Smart Array 阵列卡，其链接的 SCSI 在 linux 中识别出来的不是平常我们常认为的 sdx 设备，而是 `cciss/c0d0px` 这样的样子。

而这样的分区信息，使用 fdisk -l 是没有办法打印出来的，这个时候就要使用`sfdisk -l`命令了。这就使用到了 sfdisk 的第二个功能了。

他的第三个功能和第四个没有使用过，不过刚才看 man 手册，发现其备份和恢复分区表的方式倒是很简单。

备份分区表：

```shell
sfdisk /dev/hdd -O hdd-partition-sectors.save
```

恢复分区表：

```shell
    sfdisk /dev/hdd -I hdd-partition-sectors.save
```

下面是我的具体测试：

```shell
fdisk -l /dev/mdp0
# fdisk -l /dev/mdp0

Disk /dev/mdp0: 419 MB, 419299328 bytes
2 heads, 4 sectors/track, 102368 cylinders
Units = cylinders of 8 * 512 = 4096 bytes

Device Boot Start End Blocks Id System
/dev/mdp0p1 1 24415 97658 83 Linux
/dev/mdp0p2 24416 102368 311812 83 Linux
# ls /misc
boot.b
# df
文件系统 		1K-块 已用 可用 已用% 挂载点
/dev/mdp0p2 301959 50763 235606 18% /misc

#sfdisk /dev/mdp0 -O /tmp/mdp0-partition.save

# ls -l /tmp/mdp0-partition.save
-r–r–r– 1 root root 516 05-28 00:06 /tmp/mdp0-partition.save

# dd if=/dev/zero of=/dev/mdp0 bs=512 count=1
1+0 records in
1+0 records out
512 bytes (512 B) copied, 7.9619e-05 seconds, 6.4 MB/s

# fdisk -l /dev/mdp0

Disk /dev/mdp0: 419 MB, 419299328 bytes
2 heads, 4 sectors/track, 102368 cylinders
Units = cylinders of 8 * 512 = 4096 bytes

Disk /dev/mdp0 doesnt contain a valid partition table
# sfdisk /dev/mdp0 -I /tmp/mdp0-partition.save
Re-reading the partition table …
# fdisk -l /dev/mdp0

Disk /dev/mdp0: 419 MB, 419299328 bytes
2 heads, 4 sectors/track, 102368 cylinders
Units = cylinders of 8 * 512 = 4096 bytes

Device Boot Start End Blocks Id System
/dev/mdp0p1 1 24415 97658 83 Linux
/dev/mdp0p2 24416 102368 311812 83 Linux

# mount /dev/mdp0p2 /misc
# ls /misc
boot.b
```

从上面的例子可以看出，sfdisk 确实可以做到分区的备份和恢复，而且分区已有数据并不会丢失。  

这其实和拿着一片原始钥匙去配钥匙的地方复制一片出来，某天原始的丢了，用这片复制的打开房门，房子的摆设当然什么都没有改变。


### /sbin 下剩余的命令

从 t 到 z 开头的命令不太多，而且很多也是常用的，也只有那些基本用法，因此我就不记录到这里。

### tune2fs

这无疑是文件系统中强劲的命令了，虽然我我最常用的是修改需要做 fsck 之前的最大挂载次数。刚看了 man 手册，其实他的参数多着呢，很多文件系统元数据都可以修改，比如保留块大小，日志大小，挂载选项等。


还发现一个`-L`的参数是给文件系统设置卷表，以前我都是用 e2lable 来干这个活的，没有想到 tune2fs 也能做。看来下次可以不用需要 e2label 了。

`u` 开头的基本上都是和 udev 相关的，这方面，我贴过一个理解 udev 的文章，这里就不再贴了。

`v` 开头有两个命令，vboxd，vchange，是和 ISDN 有关的，这应该算是一段历史吧。

`w` 开头只有一个命令: wait_for_sysfs，是 udev 的 rpm 包里的，用在 udev 规则里。

`x` 开头的与 xfs 文件系统有关的，可惜目前我发现在 DC 上 xfs 支持得并不是太好。

`y` 开头只有一个命令：ypbind，是 NIS 的相关命令，这个从 Solaris 上迁移过来的东西，不知道有没有在 Linux 运行的案例。

`z` 开头的就没有了。


## /usr/sbin 下的命令


`/usr/sbin`下的命令明显比/sbin 下的命令多了很多，因此`/usr/sbin`应该是一个比较长的学习过程。


### alternatives

用来对系统中不同版本的同个软件进行管理的工具，比如系统里默认有多个 java 版本，到底使用那个版本的 java 程序，这时就需要 alternatives 来管理了。

`etc/alternatives`目录就是它的大本营，里面全是链接文件。  

这个命令网上有很多帖子详细说明了用法，那我就使用拿来主义了[LINUX alternatives howto](http://www.linuxquestions.org/linux/answers/Applications_GUI_Multimedia/LINUX_ALTERNATIVES_HOWTO)用一个实际的例子描述了 alternatives 的用法。

### am-eject

这是 am-utils 包的一个命令，am-utils 是一个 BSD 的 automount 程序，用来自动挂载一些文件系统。
从 `amd.conf` 的 man 手册来看，似乎 am-utils 自能自动挂载 NFS 的文件系统。
系统同时也带了 autofs 这个自动挂载程序，与 am-utils 不同的是，autofs 是内核级别的，它需要内核的支持，另外它支持各种文件系统的挂载。
而 am-utils 则是用户空间的，内核并不知道需要挂载哪些文件系统。分别有两篇文章详细描述了 autofs 和 am-utils 的用法，分别是
[Am-utils (4.4BSD Automounter Utilities)](http://www.am-utils.org/docs/am-utils/am-utils.html)和
[autofs-HOWTO](ftp://ftp.nrc.ca/pub/systems/linux/HOWTO/translations/zh/gb/mini/Automount.txt.gz)。

### anaconda

目前通用的 linux 系统安装向导程序，应该算是 python 的代表作吧。到网上搜索了一下，第一个找到的文章竟然是现在的同事柏生在 2005 年发表在 IBM 开发网站的文章，因此就直接拿过来了

[redhat 安装程序 anaconda分析](http://www.ibm.com/developerworks/cn/linux/l-anaconda/index.html)。

### anacron

anacron 是和 cron 相似的任务调度器，只不过它并不要求系统持续运行。它可以用来运行通常由 cron 运行的每日、每周、和每月的作业。

网上同样的详细的帖子介绍，[anacron](http://knit.ntsky.com/docs/redhat9/rhl-cg-zh_CN-9/s1-autotasks-anacron.html)

### bgpd

[Quagga](http://www.qugga.net)软件包的一个程序，用来实现 BGPv4 等路由协议的程序。
Quagga 是一个路由程序套件，它提供了 OSPFv2, OSPFv3, RIP v1 and v2, RIPng and BGP-4 等协议的实现。
用它，就可以把一个 Linux 系统改造成一个功能强大的路由器，和那些动则上十万百万的硬件路由器功能相当，对于 SMB(Small Middle Business)而言,应该是非常不错的选择。

Quagga 还包括下面一些命令：
`bgpd ospf6d ospfclient ospfd ripd ripngd watchquagga zebra`

quagga 的使用方式和使用 cisco 等网络设备厂商的设备差不多，可惜我对这些产品一窍不通，因此 quagga 这么好用的软件，对我来说，也就用处不大了，于是略过。

### bmc-config
bmc-config, bmc-info IPMI BMC 配置工具程序，IPMI(Intelligent Platform Management Interface）即智能平台管理接口是使硬件管理具备“智能化”的新一代通用接口标准。

用户可以利用 IPMI监视服务器的物理特征，如温度、电压、电扇工作状态、电源供应以及机箱入侵等。Ipmi 最大的优势在于它是独立于 CPU BIOS 和 OS的，
所以用户无论在开机还是关机的状态下，只要接通电源就可以实现对服务器的监控。Ipmi 是一种规范的标准，其中最重要的物理部件就是 BMC(Baseboard Management Controller
如图 1)，一种嵌入式管理微控制器，它相当于整个平台管理的“大脑”，通过它 ipmi可以监控各个传感器的数据并记录各种事件的日志。
详细的情况可以参考 IBM 开发网的一篇文档--[使用 ipmitool 实现 Linux系统下对服务器的 ipmi 管理](http://www.ibm.com/developerworks/cn/linux/l-ipmi/index.html)。

### brctl

网桥管理程序，用来设置，维护监控 linux 内核网桥配置信息。
说实在的我不太清楚网桥和 hub 之间到底有多少区别，所以也就不知道桥接是什么概念了
。于是我也没有从来没有使用过网桥的功能。[这里有篇文档](http://h50069.www5.hp.com/e-Delivery3/Forum/WebUI/Messages/ShowTopic.aspx?RID=6f770c6e-d7ee-4bc8-bb83-8345064ce11d)详细说明了网桥的概念，用途，局限性。

文章提到网桥适合小型较简单的网络，那我直接用一个 HUB 就行了，100 块不到就搞定了。

### chroot

指定根文件系统（/），当我第一次使用这个命令的时候，我才发现 Linux 原来如此强大，仅仅只需要这样一个简单的命令，就可以同时使用已经存在的多个系统。
比如我平常使用的系统是 Everest，他适合桌面办公，但是我在出去培训和自己联系一些有关企业应用的时候，我需要 DC5 的环境，这个时候 chroot 就帮上我的大忙了，仅仅只需要把 DC5 的根文件系统挂载上来，然后 chroot 到这个挂载点，DC5 的环境就有了，大部分情况下，和直接使用 DC5 没有差别，只是这个系统系统使用还是 Everest 的核心，因此当需要执行与核心有关的命令的时候，此时 chroot 就有点无能为力了，不过好在这样的情况不太多。Chroot，很伟大的创造！

### chpasswd

批量修改密码，这对维护拥有大量用户账号的管理员而言，无疑有帮助，虽然作为一个系统管理员，可以很简单的写一个程序来达到批量修改账号密码的目的，但是既然系统提供了这样的功能，那我们为什么不直接使用呢？
他从标注输入读取每行用户信息，每行由 `username:password` 组成。
对于 password 的格式，chpasswd 程序给出贴心的一面，可以使用明文的方式，这样就免除了我们自己去生成密文的烦扰，同时日后也知道用户的初始密码是多少。
另外一个方面，如果你从完全考虑，希望这个文件里不要出现明文的密码，那么你可以使用加密的密码填充 password 这个字段，只是使用这个 chpasswd 命令的时候，需要使用 `-e` 的参数。  

```shell
# cat /tmp/chpwd.txt
work:test oracle:test
# chpasswd  
```

应该说是非常方便的。

### dhcrelay

DHCP 服务器中转器。当你的网络有两个 VLAN，但是只在 VLAN A 里放置了一个 DHCP 服务器，那么如何让 VLAN B 的机器也能自动获得 IP 地址呢，这个时候，
你需要 dhcrelay 了。这方面的文档，网络上比较多，给出两篇：  

- [DHCP relay agent](http://www.homeweb.idv.tw/index.php?pl=249)  
- [dhcp server 可不可以跨网段来工作](http://linux.chinaunix.net/bbs/viewthread.php?action=printable&tid=904010)。

### dovecot

一个简单但是安全的 IMAP 服务器，也包括了 POP3 服务，支持 mbox 和 maildir 两种邮件存储格式。 
以前我还不知道系统自带了这么好的东西，也还简单。

### dump-acct

导出进程记账文件，默认是`/var/account/pacct`，他和 accton 命令是配合起来用的。

### dump-utmp

导出登录记账文件，默认是 `/var/log/wtmp`，其功能和 last 命令相似。

### dumpcap

dump 网络包，他抓获网络数据包并写入文件，文件格式可以是 libcap 的，也可以是 ethereal 的，当然也能是 tcpdump 的。
但是我不知道他和 tcpdump 有什么不同,Google 了一下，也没有发现什么，我觉得这不重要，Linux 下不是经常对某一个功能，有一打的命令可以实现吗？

man 手册给出了一个例子：

```shell
#dumpcap -i eth1 -a duration:60 -w output.pcap
```

表示抓取通过 eth1,60 秒内的数据包，并到文件`output.cap`。抓出的包内容大致如下：

```shell
# cat output.pcap |more
Accept: */*
Accept-Language: en-us
User-Agent: MSMSGS
Host: 207.46.106.77
Proxy-Connection: Keep-Alive
Connection: Keep-Alive
Pragma: no-cache
Content-Type: application/x-msn-messenger
Content-Length: 0

0F�
2^ english | onclick="return top.js.OpenExtLink(window,event,this)">
other
&nbsp;
```

### filefrag

Linux 下的文件系统相比 Windows 的文件系统要优越，其中一点就是不用做磁盘碎片整理，但是这并不意味着 Linux 下的磁盘读写就不会产生碎片。

filefrag 这个命令就是用来报告一个文件是否有碎片的程序。看几个例子

```shell
# filefrag /etc/passwd -v
Checking /etc/passwd
Filesystem type is: ef53
Filesystem cylinder groups is approximately 848
Blocksize of file /etc/passwd is 1024
File size of /etc/passwd is 1534 (2 blocks)
First block: 4245865
Last block: 4245866
/etc/passwd: 1 extent found

# filefrag /data/tools/db2_v9.iso
/data/tools/db2_v9.iso: 73 extents found, perfection would be 3 extents

# filefrag /data/tools/db2_v9.iso -v
Checking /data/tools/db2_v9.iso
Filesystem type is: ef53
Filesystem cylinder groups is approximately 245
Blocksize of file /data/tools/db2_v9.iso is 4096
File size of /data/tools/db2_v9.iso is 330647552 (80725 blocks)
First block: 3851518
Last block: 3954148
Discontinuity: Block 769 is at 3852400 (was 3852287)
Discontinuity: Block 811 is at 3852471 (was 3852441)
Discontinuity: Block 844 is at 3852560 (was 3852503)
Discontinuity: Block 2617 is at 3854368 (was 3854335)
Discontinuity: Block 2636 is at 3854528 (was 3854386)
Discontinuity: Block 2662 is at 3854610 (was 3854553)
......
/data/tools/db2_v9.iso: 73 extents found, perfection would be 3 extents
```

第一个文件`/etc/passwd`没有产生碎片，但是第二个文件就不同了，一共有 73 个不连贯的地方（输出省略了一些）。  

既然有显示磁盘碎片的程序，那我就想是否有整理磁盘碎片的程序呢，到网上搜索了一下，有人说系统上有 defrag 命令，但是我的系统显然没有。

还有人说，Linux 根本用不着做磁盘碎片整理。是啊，记得第一次解除 Linux 的时候，就看到有这样的话--Linux 不需要磁盘整理--不过如果你真的想解决这个其实不是问题的问题，那你可以尝试 fsck，
网上说这样可以减少文件带来的磁盘碎片，但是我现在没有做这个测试。等有机会应该试试。

这个命令的最后输出“perfection would be 3 extents”没有太明白他的意思，是不是说如果我采用某个方式，就可以把这些磁盘碎片的个数由 73 个降低到 3 个呢？用什么样的方法呢？没有找到。

### wireshark

[wireshark](http://www.wireshark.org/)：乍一看，不知道这个命令做什么用的，man 了一下手册，到网络上搜寻了一把，才知道原来它是大名鼎鼎的 Ethereal 的化身呀，ethereal 估计很多用过 Linux 的人都知道，它是世界上最流行的网络分析工具之一，这个强大的工具可以捕捉网络中的数据，并未用户提供网络和上层协议的各种信息，它具有安装简便，简单易用，功能丰富等特征。

值得一提的是，它还有一个非常容易操作的图形界面，当然如果你是 CLI 的死忠分析，也许 tshark 会成为你的最爱，tshark 是 wireshark 的 CLI 实现。

### tripwire

[tripwire](http://www.tripwire.org)：`*nix` 系统下最著名的文件系统完整性检查工具，这个软件采取的核心技术就是对每个要监控的文件产生一个数字签名并保留下来。

当数字签名不一致时，必定文件被修改过。当你要为你的系统安全着想的时候，你应该想起它！

### vipw

不是多么神奇和高深的命令，不带任何参数，运行后，直接打开 `/etc/passwd`，于是我知道这个命令的意思了`vipw = vi /etc/passwd`，以为是一个脚本，然后 vipw 是一个地地道道的 ELF 文件，实际上他是 VIM 的 tiny 版的实现。  
不晓得作者写这个程序的目的是什么？仅仅是为了方便编辑`/etc/passwd`？还是说万一系统没有安装编辑器，我还用它当做编辑器用，  


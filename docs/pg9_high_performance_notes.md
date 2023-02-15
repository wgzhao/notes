# 《PostgreSQL 9 High Performance》读书笔记

这是很早前阅读 [《PostgreSQL 9 High Performance》](https://www.packtpub.com/product/postgresql-90-high-performance/9781849510301) 做的一些笔记

分章节记录

## 数据库版本

这章主要介绍了 PostgreSQL 的历史，感兴趣的，可以看看，和技术关系不大

## 硬件选型

1.  不用 SSD 硬盘的三个主要原因

    1.  当前硬盘容量不大，而数据库一向都占用磁盘较多
    2.  存储这块的费用会相当高
    3.  大部分 SSD 设计时并没有很好的考虑写缓存(write-cache)

2.  write-back cache 方式并不是适合用于存储数据的文件系统

3.  磁盘阵列一般都用 wirte-back cache，因为它配置了电池，一般称为
    battery-backed write cache(BBC or BBWC)

4.  几条预防 write-back cache 出故障的基本法则

    1.  确保你使用的文件系统完整实现了 fsync 调用或者类似机制的功能

    2.  监控磁盘控制器电池。有些控制卡自身会监控其状态，如果发现掉电或者工作不正常时，会从 write-back
        改成 write-through 模式，当然，其性能会下降

    3.  禁止任何磁盘的 write-cache 模式。大部分 RAID 卡会来做这点，他们会优先使用
        BBC 模式

5.  磁盘转速以及每秒提交的写速关系\

    转速(RPM) 转时(ms) 最大提交数(/s)

    ***

         5400        11.1           90
         7200        8.3           120
         10000       6.0           166
         15000       4.0           250

    上述图表关系意味着一个普通的 7200 转的硬盘，单个客户端提交的事务不可能超过 120/sec.

## 硬件性能测试

### 内存性能测试

- **memtest86+** 大部分 Linux 发型版本都自带了 memtest86+内存测试工具，如果没有，也可以从<http://www.memtest.org>下载，然后创建一个包含 memtest86+引导 CD 进行测试

- **STREAM 内存测试** STEAM 是一个内存带宽测试程序，可以从<http://www.cs.virginia.edu/stream>下载，网站上也有很多测试报告事例.\

### CPU 性能测试

直接使用数据库来做一些查询操作就是测试 CPU 性能的最好工具，在 psql 里开启\\timing 功能，利用*generate_series*函数来创建巨量的数列，便能看出 CPU 的性能。比如类似下面的一些指令：

```sql
postgres=#\timing
postgres=#SELECT sum(generate_series) FROM generate_series(1,1000000);
postgres=#CREATE TABLE test (id INTEGER PRIMARY KEY);
INSERT INTO test VALUES (generate series(1,100000));
postgres=#EXPLAIN ANALYZE SELECT COUNT(*) FROM test;
```

### 内存和 CPU 慢的根源

大部分内存设置为双通道模式，一对内存应该插入到特定的插槽内，如果安装不正确，内存将会运行在单通道模式。memtest86+程序有次提示，另外 BIOS 里也能侦测到这种情况\
劣质的内存颗粒会严重拖累系统性能，因为主板并不能充分利用内存所宣称的高速。大部分系统的缺省配置都是保守的，它一般会从内存提供的 SPD（Serial
Presence
Detect）信息中查询它应该运行的速度，但这往往不是其最高性能，因此需要人工调整内存时钟。\
高性能的内存使用了称为"Extreme Memory
Profile(XMP)\"协议，它可以在系统启动时向 BIOS 传递它可以运行的最高速率。但是 BIOS 缺省情况下并不进检查和使用 XMP，这就使得你的内存看上去没有并没有那么快。\
另外一个原因就是内存和处理器的时钟频率不能很好的合作，一方面需要注意主板和 BIOS 的设置，以保证 CPU 和内存的时钟频率能发挥最大功效。另外一个方面就是当前大部分操作系统为了节省，都有 CPU 调速的功能，在空闲时，会将 CPU 的主频调的很低，比如一个 2.4GHz 的 CPU，在空闲时调整为 1GHz 是很常见的事情，如果时 Linux 系统，可以查看/proc/cpuinfo 来获取当前 CPU 的运行频率。在测试性能时，应该关闭 CPU 调速的功能。

### 磁盘性能

#### 随机访问及 IOPS

IOPS(Input/Outpu Per Second)
是衡量一个磁盘性能的重要参数，它综合衡量了磁盘随机读寻道时间，平均延迟时间等影响性能的因素。

根据磁盘常见对磁盘给出的参数，我们可以大致算出一个磁盘的 IOPS。比如 seagate 出品的 Momentus
7200.4 桌面型磁盘规格参数如下：

转速：7200 RPM

平均延迟：4.17ms

随机读寻道时间：11.0ms

实际上通过转速，我们就能大致计算出来平均延迟，比如 7200 转的磁盘，就意味着，转一圈需要$1*60/7200 = 8.33ms$。
最差的情况就是你每次等待都需要磁盘转一圈，但这种几率不大，大部分等待都不想要等待一圈，因此我们取算术平均，每次等待传半圈的时间，也就是$8.33/2 = 4.17ms$。
由此看出，所有 7200
RPM 的磁盘，其平均延迟都应该是一样的，但是随机读取寻道时间则依赖于磁盘的大小、质量以及其他因素了。

平均延迟时间 $RL = 1/ RPM / 60 / 2 = 4.17 ms$

寻道时间 $S = 11.0 ms$

$IOPS =  1 / (RL + S)  = 1 / (4.17ms + 11ms) = 65.9 IOPS$\

#### 顺序访问及 ZCAV

CAV(Constant Anular
Velocity)，就恒定角速度。指的是磁盘在任意时刻，转速是一致的，但是因为磁盘的盘面是一个圆形，这就意味着在相同时间内，越靠近磁盘外层，扫过的面积的也就越大。这也就是我们常说的，磁盘外道的读取速度要快于磁盘内道。

有关 CAV，以及相关工具的讨论，可以参考下面的链接：
<http://www.coker.com.au/bonnie++/zcav/>

#### 短冲程(short stroking)

我们知道磁盘的外道读写速度要快，因此有专门的技术来保证你的数据只存放在外道上，这就是成为 Short
stroking 的技术。

#### 磁盘性能测试工具

Windows 下使用[hdtune](http://www.hdtune.com)免费版即可\
Unix 下，系统再带的 dd 指令就可以作为磁盘性能测试的基本工具。一般 dd 测试读写的文件大小至少是物理内存的两倍，以保证系统不会把实际需要写入磁盘的数据缓存到内存上。考虑到 postgresql 默认的块大小是 8KB，因此 dd 测试需要的块大小为：

$blocks = 250,000 * (gigabytes of RAM)$

测试方法一般如下：

```shell
time sh -c "dd if=/dev/zero of=bigfile bs=8k count=blocks && sync"
time dd if=bigfile of=/dev/null bs=8k
```

Unix 下也可以使用开源的 bonnie++工具\
1）Bonnie++

当前最新的版本是 1.96，也就是所谓的 2.0 版本

```shell
bonnie++ -f -n 0 | tee `hostname`.bonnie
cat `hostname`.bonnie |grep "," | bon_csv2html >`hostname`.htm
```

-n 0 表示跳过文件创建测试

-f 表示快速模式，跳过每字符的 I/0 测试

2）bonnie++ zcav

zcav 是 Bonnie 项目中独立出来的程序，用于跟踪整个磁盘的传输率，可以利用 gnuplot 来绘制出漂亮的图形，示例如下：

```shell
zcav -lsda.zcav -f /dev/sda
zcav -lsdb.zcav -f /dev/sdb
```

上述指令需要 root 权限，将会产生两个日志文件 sda.zcav 和 sdb.zcav

而后，我们通过下面的指令生成图形

```gnuplot
unset autoscale x
set autoscale xmax
unset autoscale y
set autoscale ymax
set xlabel "Position MB"
set ylabel "KB/s"
set key right bottom
plot "sda.zcav" title "Dell E6410 laptop", "sub" title "Huawei SSD"
set terminal png
set output "disks.png"
replot
```

sysbench 是一个开源的数据库测试工具，它不仅可以测试数据库本身的性能，也能通过数据库来做硬件测试，下面给出一些测试例子：\

`sysbench --test=cpu run`\

```shell
#!/bin/sh
THREADS=1
GBSIZE=4
MODE=rndrd
sysbench --test=fileio --num-threads=$THREADS --file-num=$GBSIZE \
--file-total-size=${GBSIZE}G --file-block-size=8K --file-test-mode=${MODE} \
--file-fsync-freq=0 --file-fsync-end=no prepare

sysbench --test=fileio --num-threads=$THREADS --file-num=$GBSIZE \
--file-total-size=${GBSIZE}G --file-block-size=8K --file-test-mode=${MODE} \
--file-fsync-freq=0 --file-fsync-end=no run --max-time=60

sysbench --test=fileio --num-threads=$THREADS --file-num=$GBSIZE \
--file-total-size=${GBSIZE}G --file-block-size=8K --file-test-mode=${MODE} \
--file-fsync-freq=0 --file-fsync-end=no cleanup
```

测试分为 3 个步骤，首先创建测试文件，然后测试运行，最后清理测试文件。
测试中，我们可以调整测试文件的大小，测试运行的线程数以及采取何种读写模式。\

```shell
#!/bin/sh
DRIVE="/dev/sda"
## Disable write cache
hdparm -W 0 $DRIVE
echo fsync with write cache disabled, look for "Requests/sec"
sysbench --test=fileio --file-fsync-freq=1 --file-num=1 --file-total-size=16384
--file-test-mode=rndwr run
## Enable write cache (returning it to the usual default)
hdparm -W 1 $DRIVE
echo fsync with write cache enabled, look for "Requests/sec"
sysbench --test=fileio --file-fsync-freq=1 --file-num=1 --file-total-size=16384
--file-test-mode=rndwr run
```

#### 其他复杂的磁盘性能测试工具

1.  [iozone](http://www.iozone.org) 可以测试各种磁盘使用场景

2.  [fio](http://freshmeat.net/projects/fio/)
    通过脚本来构建和测试各种你想测试的场景，可以看这个例子：
    <http://wiki.postgresql.org/wiki/HP_ProLiant_DL380_G5_Tuning_Guide>

3.  [pgiosim](http://pgfoundry.org/projects/pgiosim/)

## 磁盘设置

### 文件系统

#### ext2

Linux 上原生的最古老的文件系统，没有日志，因此大部分场景下都不适合用该文件系统。

不过因为没有日志，所得它的读写速度很快，有时，我们可以把 ext2 文件系统用在 PostgreSQL
WAL 上。

但是要注意系统重启后，自动执行 fsck 所带来的风险。因此只有明确 WAL 成为瓶颈时，可以考上使用更快但是有一定风险的 ext2 文件系统。

#### ext3

ext3 在 ext2 的基础上增加了日志功能，它向后兼容 ext2。而且 ext2 和 ext3 两种文件系统之间可以互转。

目前，在 ext3 上，有三种写日志的方法，它可以通过挂载时传递 data 参数来指定使用哪种方式：

- **data=writeback**
  数据改变不会记录在日志。元数据改变记录在日志，不保证数据块写入的相关顺序。当崩溃后，文件可以恢复，不过文件结尾可能有些无用的数据，另外，文件的内容可能是新旧数据混合.

- **data=ordered**
  元数据记录在日志，数据改变不进日志。只有当相关联的数据写入磁盘，才写元数据，也是\"ordered\"的由来。

- **data=journal** 数据和元数据的改变都写入日志

相比 WAL 存储在 ext2 上获得性能的优势，ext3 +
data=writeback 是一个不错的替换选择，一来性能不会太差，而来有日志的保证，崩溃后恢复时间大大缩短。只是要考虑文件膨胀的问题。

#### ext4

ext4 真正开始能在产品中使用是从 kernel2.6.28 开始，不过因为一些调用 fsync 处理的 bug，对于 PostgreSQL 而言，kernel
\> 2.6.32 的可以考虑使用 ext4。也就是 RedHat 6 和 Ubuntu 10.04 可以开始用了。

尽管理论上，ext4 不存在之前 ext3 文件系统 16TB 的大小限制，但是 mkfs 工具目前依然还是有这个限制。

从 PostgreSQL 的角度来看，ext4 相比 ext3 的主要提升在于能够更好的处理 write
barriers 和 fsync 操作。

#### XFS

XFS 是 SGI 设计的一款高效率日志文件系统，它比 ext3 速度稍快，从原理上来讲，它更像是 ext3+data=writeback 运行模式。因此如果想使用 XFS,要记得设置 PostgreSQL 的*full_page_writes*参数

下面是一个 XFS 和 ext3 的速度比较表，可以大致看出 XFS 的优势

      Filesystem       Sequential write(MB/s)   Sequential read(MB/s)

---

ext3 data=ordered 39-58 44-72
ext3 data=journal 25-30 49067
XFS 68-72 72-77

RHEL5 初步开始支持 XFS，RHEL6 已经完全支持 XFS。因此在特别大的磁盘空间上，比如希望文件系统超过 16TB 的情况下，推荐使用 XFS，而且文件系统越大，XFS 相比 ext3/ext4 越有优势。

缺省情况下，XFS 使用 write barriers，XFS 也偏执的认为易失缓存(volatile
write
cache)的磁盘会丢失数据。为了防止这点，XFS 总是积极的给磁盘发送刷盘(flush)指令。因此，如果你有非易失缓存设备，比如带后备电池的写控制器，那么缓存刷盘就是浪费。在这种情况下，挂载 XFS 推荐使用 nobarriers 选项来确保禁止缓存刷盘。

#### 其他 Linux 文件系统

JFS

: 和 XFS 类似，只是 CPU 使用更少，目前还没有得到主流发型版本的完整支持。

ReiserFS

: 作为第一个进入 Linux
kernel 的日志文件系统，而且 SuSE 还把它作为缺省的文件系统，但是随着作者的被捕，其前景堪忧。

Btrfs

: Oracle 捐赠的文件系统，被认为是未来 Linux 的主要文件系统，目前还没有到稳定版，基于它的一些独一特性，比如快照，校验和。它将来很有可能是极具竞争力的文件系统。

#### Solaris UFS

Solaris
UFS 用于 PostgreSQL 并不太好。一个主要的问题是它仅能缓存小文件，而数据库期望的是能缓存大文件。另外，能用于缓存的内存大小很小，对于 SPARC 系统而言，只能用到内存的 12%。而 Intel/AMD
x64 只只有可怜的 64MB。对于最后这个问题，我们可以调整参数来修复，一个是掉正类似预读取的大小，一个是类似读写到交换空间的大小

    set maxphys=1048576
    set klustsize=1048576

在 SPARC Solaris 系统上，还可以调整下面的两个参数：

    set freebebind=0
    set segmap_percent=60

对于 Intel/AMD x64 Solaris 系统，则设置下面的两个参数：

    set ufs:freebehind=0
    set segmapsize=1073741824

#### FreeBSD UFS2

和 Linux 类似，通过修改 read-ahead 能显著提升顺序读的速度。参数的单位是文件系统块大小，默认是 8KB。
这个参数我们可以设置在 32 到 256 之间，通过修改/etc/sysctl.conf 文件，增加类似下面的这行：\
vfs.read_max = 32\
然后通过下面的指令，使其生效：\
/etc/rc.d/sysctl start

#### ZFS

很少有文件系统能像 ZFS 一样拥有众多的拥护者，当然它的很多先进特性，特别是自带的卷管理和实时快照功能我们在很多地方都能看到其介绍。
ZFS 默认情况下使用 128KB 大小的记录块，对 PostgreSQL 而言，这个数有点大，反而使得效率不高，我们可以通过下面的设置来使得文件系统的块大小和 PostgreSQL 的块大小一致：\
`zfs set recordsize=8K zp1/data`\
这个大小对 WAL 并不是优化值，相反，其缺省的 128KB 应该更好。ZFS 尽可能消耗掉所有的内存用于它的自适应可替换缓存(Adaptive
Replacement Cache aka
ARC)，对于 PostgreSQL 而言，需要降低该值，不同的 Soloris 发型版本，其调整 ARC 的方式有所不同，具体的可以参考<http://www.solarisinternals.com/wiki/index.php/ZFS_Evil_Tuning_Guide>文章里的\"Limiting
the ARC Cache\"章节。

对于 FreeBSD 系统而言，可以参考<http://wiki.freebsd.org/ZFSTuningGuide>，这片文档里提到了一个*arc_summary.pl*的有用脚本，它可以在 FreeBSD 和 Solari 上运行。

ZFS 使用一种称为\"intent
log\"的结构来处理日志，在具有多个磁盘的高性能系统上，针对数据库磁盘，一般对分配出专门用来存储\"intent
log\"的存储池。而对于用于存储 WAL 的磁盘，就不需要专门的存储池来存储\"intent
log"。有关更具体的如何为数据库优化 ZFS 的指南，可以参考下面这个链接：\
<http://www.solarisinternals.com/wiki/index.php/ZFS_for_Databases>

和 XFS 相似，如果你的系统有非易失写缓存，ZFS 提供的缓存刷盘会影响性能，我们可以通过设置 zfs_nocacheflush 参数来禁止它。在/etc/system 里增加下面的这行：\
`set zfs:zfs_nocacheflush=1`\
因为 ZFS 的\"intent
log\"的健壮性以及块校验和的特性，当它用于 PostgreSQL 时，可以禁止 full_page_writes 参数来获得性能提升，且少有风险。

#### FAT32

目前已经不推荐在数据库上使用 FAT32 文件系统上了，无日志、非正常关机需要执行 chkdsk 恢复。基于这些原因，PostgreSQL 安装程序不支持把数据库集群直接安装到 FAT32 系统上。当然，你手工执行 initdb 还是可行的。

#### NTFS

NTFS 是微软的旗舰文件系统，它使用非有序元数据日志记录。有点类似 Linux 下的 writeback 日志模式。大部分情况下，它值得信赖，但在一些罕见的情况下，有可能必须对文件系统执行 chkdsk 指令才能挂载上来。就像较早的 PostgreSQL 安装程序所描述的那样，当使用 NTFS 是，偏向于使用下面的数据库配置参数：

`open_datasync=fsync_writethrough`\
NTFS 使用称为"filesystem
junction\"的功能来允许在其上创建功能等价的符号链接，Junction 工具可以从<http://technet.microsoft.com/en-us/sysinternals/bb896768>下载。这就使得我们可以在 NTFS 卷上保留目录结构不变的情况下把 pg_xlog 存放到另外一个文件系统上。这个功能也可以使用在表空间上。

和类似 UNIX 的 noatime 参数相似，我们也可以在 NTFS 卷上关闭对应的功能。\
`fsutil behavior set disablelastaccess 1`\
因为历史原因，NTFS 最自动生成符合 8.3 格式的文件名，以便向后兼容，我们可以通过下面的参数禁止这个功能：\
`fsutil behavior set disable8dot3 1`\
要注意的是，如果你设置了 disable8to3 参数，可能有有些意外的情况发生，比如如果你的%TEMP%或%TMP%环境变量使用了短名来访问文件或目录，关闭这个参数后，会导致目录找不到，因此你需要更新所有使用短名的环境变量。

#### Write barriers

绝大部分磁盘都有易失缓存，因此 PostgreSQL 的 WAL 写模式与此并不兼容，这里最重要的一个要求就是当一个文件同步操作发生时（fsync)，文件系统必须确保所有相关数据都写入非易失缓存或磁盘本身。
文件系统元数据写操作也有同样的要求，对数据更新的写操作要求日志更新首先按照正确的顺序写入安全区域。

Linux Kernel 为了支持这些场景，开发称之为写屏障(write
barrios)的功能。内核文档是如此描述 write barries 的：

> All requests queued before a barrier request must be finished (made it
> to the physical medium) before the barrier request is started, and all
> requests queued after the barrier request must be started only after
> the barrier request is finished (again, made it to the physical
> medium).

意思是当一个 barrier 请求到来时，所有在这个之前的请求队列比如完成（写入物理介质），然后这个 barrier 请求才能响应，而 barries 之后的所有请求队列只有当这个 barries 完成后（写入物理介质）才能开始。

这相当于在一个较长的请求队列里强行插入一个栅栏，只有栅栏前的队列都完成，后面的才能发起。可以想象类似超市排队结账时的场景，当队列过长时，保安会过来干涉，它站在队伍的中间，然后说，"后面的稍微等一下，为了防止拥挤，等前面的都结算完后，你们再过去"。

#### 磁盘对 barriers 的支持

SCSI/SAS 磁盘允许读写操作指定强制单元访问(Force Unit Access aka
FUA)，它可以不通过缓存直接访问磁盘介质，也就是我们常说的写透(write-through)。同时也支持 SYNCHRONIZE
CACHE 调用来刷盘。

早期的 IDE 磁盘实现了称之为 FLUSH
CACHE 的系统调用，不过磁盘大小限制为 137GB。ATA-6 规范中增加了对大磁盘的支持，并增加了新的 FLUSH
CACHE EXT 系统调用。

支持原生命令队列（Native Command
Queuing)的 SATA 磁盘也能处理 FUA。Linux 从 2.6.19 内核开始对其 NCQ 的支持放在了 libata 驱动力，不过有一些发型版本（比如 RedHat）为了向后兼容，还是把对 NCQ 的支持驱动放在内核里。

#### 文件系统对 barriers 的支持

ext3 文件系统如果只使用简单卷，理论上是支持 barriers 的。如果使用软 RAID 或者 LVM，则 barriers 根本无效。

XFS 能够正确处理 write barriers，而且缺省情况下是启动这个功能的。

ext4 同样支持 barriers，缺省情况下，也是启动此功能的。

### 通用 Linux 文件系统调优方案

#### Read-ahead

将 read-ahead 可调范围一般为 4096 16384，缺省值一般是 256，可以通过下面的指令查看缺省值

`blockdev--getra /dev/sda `

设置 read-ahead 的指令如下：

`blockdev --setra 4096 /dev/sda`

#### File access times

关闭 atime，在 fstab 里对应的设备挂载选项上加上 noatime，类似下面这样：

`/dev/sda1     /     ext3     noatime,errors=remount-ro 0 1`

#### Read caching and swapping

设置 vm.swappiness=0，表示内核尽可能使用文件系统 cache 而不是 swap

关闭 overcommit 行为，也就是设置 vm.overcommit_memory=2

#### write caching size

有两个参数可以调整上述行为

**/proc/sys/vm/dirty_background_ratio**: 这
个参数控制文件系统的 pdflush 进程，在何时刷新磁盘。单位是百分比，表示系统内存的百分比，意思是当写缓冲使用到系统内存多少的时候，pdflush 开始向磁盘写出数据。增大之会使用更多系统内存用于磁盘写缓冲，也可以极大提高系统的写性能。但是，当你需要持续、恒定的写入场合时，应该降低其数值，一般启动上缺省是
5。

**/proc/sys/vm/dirty_ratio**:
这个参数控制文件系统的文件系统写缓冲区的大小，单位是百分比，表示系统内存的百分比，表示当写缓冲使用到系统内存多少的时候，开始向磁盘写出数据。增大之会使用更多系统内存用于磁盘写缓冲，也可以极大提高系统的写性能。但是，当你需要持续、恒定的写入场合时，应该降低其数值，一般启动上缺省是
10

对于超过 8G 的内存服务器，可以考虑将 dirty_background_ratio
设置为 1，把 dirty_ratio 设置为 2

#### I/O 调度算法

elevator=cfq

: 完全公平队列，它试图把 I/O 带宽平分给所有的请求。这是 Linux 默认调度算法

elevator=deadline

: 目的在于平衡 I/O 和请求时间，确保不会出现因等待时间太长而"饿死"的请求。

elevator=noop

: 不做任何复杂的调度，它只是简单的处理块请求合并，并依据底层设备顺序对请求排序。

elevator=as

: Anticipatory 调度有意延迟 I/O 请求，以期能合并一些请求，而从可以批处理。

目前四种调度算法，其实对数据库的影响不如上述一些参数，比如如果你想提升读的性能，那么考虑调整 read
ahead 参数，如果想提升写的性能，调整 write chaching size。

下面描述一种调度算法影响系统效率的例子：

对于有大量磁盘读写的系统，比如 RAID 或者 SAN，使用 noop 调度算法可以快速的把请求数据转发给硬件，如果硬件有大 cache 的话，是可以提升性能

对于桌面系统，磁盘 I/O 能力有限，as 调度算法能够更好的排序读写请求到批处理里，不过很显然，不适合典型的数据库服务。

### 文件系统方面的调优总结

1.  当前写日志是大部分文件系统用于崩溃后恢复的一种机制

2.  对于 Linux，ext3 文件存在更多的可调优的可能性。XFS 和 ext4 目前还算是在产品早期阶段，但相信将来是 ext3 的一个很好的替代方案

3.  Solaris 和 FreeBSD
    都有较老版本的 UFS 文件系统和较新的 ZFS 文件系统。ZFS 的一些特性使得它很适合大型数据库

4.  Windows 的 NTFS 借鉴了很多 UNIX 上文件系统的特性

5.  增加 read-ahead，停止 atime 的更新，调整用于写缓存的数值都是通用的调优技巧，而且通常都能提高数据库的性能

6.  PostgreSQL 可以将数据迁移位置，方法是通过操作系统的符号链接或者创建表空间

7.  是否需要将数据库的各组成部分分不到不同的磁盘上，很大程度上取决于应用程序和使用方式

### 针对 PostgreSQL 的磁盘布局

因为 PostgreSQL 对所有的文件都使用标准文件系统，数据库的有几部分我们可以重新分配路径，有一些通过直接移动相关文件，有些则通过创建符号链接来实现。

#### 符号链接

我们可以把 WAL 存放到独立的特别为其优化一个文件系统上，然后通过符号链接来保持其数据库目录结构不变：

```shell
$ cd $PGDATA
$ mv pg_xlog /disk
$ ln -s /disk/pg_xlog pg_xlog
```

从 PostgreSQL
8.3 以后，在执行 initdb 时，我们可以通过指定\--xlogdir 参数来指定 xlog 的存放位置，不过其工作原来和上一致，也是创建符号链接。

#### 表空间

我们可以通过创建表空间时设置 location 参数来指定其表空间内的数据文件和表存到到不同的位置：

```text
$ mkdir /disk/pgdata
$ psql
postgres=# CREATE TABLESPACE disk LOCATION '/disk/pgdata';
postgres=# CREATE TABLE t(i int) TABLESPACE disk;
```

在数据库内部，这个功能也是通过符号链接来实现的，因此上述这些操作都需要文件系统的支持。

#### 数据库目录树

通过 initdb 指令，PostgreSQL 创建了下面的主要目录结构，分别描述如下：

base

: 这是存储 pg_default 的位置，包括缺省的表空间、模板数据库以及任何不通过显示指定位置的表空间文件存放位置。

global

: 存放 pg_global 里设置的虚拟表空间，这里是在一个集群里能被所有数据库共享的系统表，比如数据库角色和其他系统条目

pg_clog

: 事务提交日志存放点。这些数据被 VACUUM 指令读取，因此这些文件不再需要，就被删除。如果 pg_clog 所在文件系统在挂载时绕过了文件系统缓存，则它会称为严重拖累数据库性能的罪魁祸首。

pg_stat_tmp

: 这个目录用于存储那些用于数据库统计的文件。这些文件都不会太大。

pg_tblspc

: 当创建一个新的表空间时，会在这个目录创建一个指向表空间目录的符号链接文件。删除表空间时，则删除对应的符号链接文件。

pg_xlog

: 存放 WAL 用于崩溃后回恢复的文件，也就是重做日志

pg_subtrans

: 存放与子事务相关的数据

pg_multixact

: 存放多事务状态数据。当你的应用程序大量使用共享行级锁时，这个目录就会被大量使用

pg_twophase

: 存放和两阶段提交相关的数据

#### 临时文件

PostgreSQL 有几个需要保存临时文件的应用场景，一个是使用 create temporary
table 来创建表以及对应的索引；一个是当一个查询里包含排序操作时，其数据超过了 work_mem 设置的大小。
默认情况下，临时对象存放在 base/pgsql_tmp 目录，除非重新指定了位置。

#### 磁盘组、RAID 以及磁盘布局

如果你磁盘很多，而且磁盘空间利用不是问题，那么下面的一个布局对提升性能还是很有帮助的：\

       Location       Disks   RAID Level       Purpose

---

       /(root)          2         1              OS
       \$PGDATA        6+         10          Database

\$PGDATA/pg_xlog 2 1 WAL
Tablespace 1+ None Temporary files

\
针对上面的磁盘布局模式，然后考虑到不同组件对缓存刷盘的需求不同，我们可以考虑下面的 cache 设置方式：\

      Function       Cache Flushes           Access Pattern

---

         OS              Rare         Mix of sequential and random
      Database         Regularly      Mix of sequential and random
         WAL           Constant                Sequential

Temporary files Never More random as client increases

### 磁盘布局指南

- 避免把 WAL 和操作系统放在一个磁盘上，他们是完全不同的访问模式

- 如果你能确保你的数据库不会有大数据量排序，你可以让临时文件存放在默认的位置

- 在使用 ext3 的 Linux 系统上，尽可能把 WAL 存放到另外的磁盘上，不要和数据库文件搅和在一个磁盘上

- 文件系统绝大部分都是通过写日志来保证崩溃后可以恢复，日志虽然增加了开销，但使得恢复时间可以在预期之内

- 对于 Linux 系统，虽然 ext3 可以满足各类场景，但并不是最佳选择。虽然 XFS 和 ext4 还没有达到产品级质量，但是他们应该很快可以替代 ext3(我觉得 ext4 已经很稳定了呀）

- Solaris 和 FreeBSD 都拥有老的 UFS 文件系统和更新的 ZFS 文件系统。ZFS 的一些特性使得它称为大数据的唯一合适选择

- Windows 平台的 NTFS 借鉴了很多基于 UNIX 系统上的文件系统的特性，使得其称为可信赖的文件系统

- 提高预读(read-ahead)值，禁止更新文件访问时间戳，调整用于 cache 的内存数量，这些技巧都可以获得更好的性能

- PostgreSQL 允许通过符号链接和创建表空间来重新分配数据某些部件的路径。

## 用于数据库缓存的内存

这章主要是讨论 shared_buffers 参数设置的内存是如何使用的，通过监控和优化来这部分内存来获得高性能，这也是 PostgreSQL 里最重要的性能参数。\

### postgresql.conf 里的内存单元

早期的 PostgreSQL 版本，postgresql.conf 文件里的关于内存的设置参数的设置，你需要知道每一个参数的单位。有的可能是 1KB，有的可能是 8KB。\
从 PostgreSQL
8.2 以后，关于内存数值的设置就简化了，你可以直接设置带单位的数值，比如假定你设置下面的参数：\
`wal_buffers = 64KB`\
你登陆数据库，通过 show 指令，可以看到 wal_buffers 的大小就是你所设置的参数：

```sql
wgzhao=# show wal_buffers;
 wal_buffers
-------------
 64kB
(1 row)
```

不过 PostgreSQL 内部还是会把参数值转化为它自己内部单位，比如 wal_buffers 的内部单位是块大小(8k)，因此，在 PostgreSQL 看来，wal_buffers 的值其实是 8。
我们可以通过下面的查询语句可以证实：

```sql
wgzhao=# select name,setting,unit,current_setting(name)
from pg_settings where name = 'wal_buffers';
    name     | setting | unit | current_setting
-------------+---------+------+-----------------
 wal_buffers | 8       | 8kB  | 64kB
(1 row)
```

### 为大缓存增加 UNIX 共享内存参数值

initdb 默认设置的 shared_buffers 一般都很小，这是考虑到 initdb 需要在各种平台保证执行成功，而很多 UNIX 系统，默认情况下允许分配的内存都很小，一般不超过 32MB。
所以 initdb 所设置的 shared_buffers 不适合任何场景，它仅仅只是保证执行成功，以及数据库服务能够启动。\
[这篇文章](http://www.postgresql.org/docs/current/static/server-start.html)里描述了各种于共享内存相关启动错误信息，其中一种可能的错误类似下面这样:

    FATAL: could not create shared memory segment: Invalid argument
    DETAIL: Failed system call was shmget(key=5440001, size=4011376640, 03600)

[这篇文档](http://www.postgresql.org/ docs/current/static/kernel-resources.html)描述了绝大部分系统上如何调整和 shared_buffers 相关的内核参数.\
对于支持 getconf 指令的 UNIX 系统，我们可以通过下面的一个脚本（假设名为*shmsetup.sh*)来提高可分配内存的大小.\

```shell
#!/bin/bash
## simple shmsetup script
page_size=`getconf PAGE_SIZE`
phys_pages=`getconf _PHYS_PAGES`
shmall=`expr $phys_pages / 2`
shmmax=`expr $shmall \* $page_size`
echo kernel.shmmax = $shmmax
echo kernel.shmall =$shmall
```

该脚本设置最大共享块大小为物理内存的一半。对于 Linux 系统，我们把上述脚本的输出写入/etc/sysctl.conf\
`$ sudo ./shmsetup.sh >>/etc/sysctl.conf`

### 内核信号量

除了共享内存外，另外一个需要调整的参数是系统可有效使用的信号量数。缺省的 Linux 系统看上去大概是这个样子：\

    $ ipcs -l
    ...
    ------ Semaphore Limits --------
    max number of arrays = 128
    max semaphores per array = 250
    max semaphores system wide = 32000
    max ops per semop call = 32
    semaphore max value = 32767

    ...

通过查询 kernel.sem 参数也能获得上述值，比如：

```shell
## sysctl kernel.sem
kernel.sem = 250 32000 32 128
```

### 评估共享内存分配

我们可以大致预测 PostgreSQL 需要使用多少内存，[这篇文档](http://www.postgresql.org/docs/current/static/kernel-resources.html)
结尾处的一个表格给出了一个大致的内存消耗评估表，摘录如下：\

用途 大致需要的内存(v8.3)

---

Connections (1800 + 270 \* max_locks_per_transaction) \* max_connections
Autovacuum workers (1800 + 270 \* max_locks_per_transaction) \* autovacuum_max_works
Prepared transactions (770 + 270 \* max_locks_per_transaction) \* max_prepared_transactions
Shared disk buffers (block_size + 208) \* shared_buffers
WAL buffers (wal_block_size + 8) \* wal_buffers
Fixed space requirements 770 KB

\
默认情况下，PostgreSQL 设置的上述参数大致是：\

参数 缺省值

---

max_locks_per_transaction 64  
 max_connections 100  
 autovacuum_max_works 3  
 max_prepared_transactions 0  
 block_size 8192  
 wal_block_size 8192  
 wal_buffers 8

\
通过这些参数，我们就可以简化第一个表格的结果，类似下面这样：\

用途 大致内存

---

Connections + AV workers 1.9MB  
 Shared disk buffers 8400 \* shared_buffers  
 WAL buffers 64 KB  
 Fixed space requirements 770 KB

### 检验数据库缓存

我们可以通过 pg_buffercache 模块来检验 shared_buffers 设置的缓存里保存的内容。
有关 pg_buffercache 的介绍可以参考这个链接：<http://www.postgresql.org/docs/current/static/pgbuffercache.html>\
一旦安装了 pg_buffercache 模块，我们就可以查看当数据库活动时，内存里的块是如何改变的。这是理解数据库相关工作原理的最佳方法。
在生产服务器上，也许 pg_buffercache 模块不会显得至关重要，但是对于我们学习数据库如何利用共享内存是有极大帮助的。

### 安装 pg_buffercache 模块

pg_buffercache 由 C 语言编写的动态库和一组 sql 语句组成。对每个你需要观察的数据库，都需要安装该模块(导入其 sql 语句)。方法类似下面这样：

```sql
createdb pgbench
psql -d pgbench -f /usr/share/postgresql/contrib/pg_buffercache.sql
SET
CREATE FUNCTION
CREATE VIEW
REVOKE
REVOKE
```

我们可以通过查询 shared_buffers 值来验证 pg_buffercache 安装没有问题，就像下面这样：

```sql
pgbench=# select name,setting,unit,current_setting(name)
from pg_settings where name = 'shared_buffers';
      name      | setting | unit | current_setting
----------------+---------+------+-----------------
 shared_buffers | 65536   | 8kB  | 512MB
(1 row)

pgbench=# select count(*) from pg_buffercache;
 count
-------
 65536
(1 row)
```

这一章的最后，我们还会利用该模块给出一些有用的报表

### 数据库缓存 vs 操作系统缓存

和其他传统的数据库产品不同，PostgreSQL 不采取甚至不偏向将内存的大部分划为自己所用。数据库的大部分读写都使用了标准的系统条调用，因此它允许操作系统的缓存按照自己的方式来使用。在某些 WAL 的配置中，如果绕过了操作系统的缓存，那会是个问题。\
如果你以前使用的数据库是的配置策略是把绝大部分内存划给数据库，且数据库的读写操作通过同步(synchronouse)或直接 IO(direct-io)方式绕过操作系统缓存。那么这种策略不要用在 PostgreSQL 上。

### 重复的缓存数据

shared_buffers 不要设置得太大的其中一个原因是操作系统缓存也被用于读写操作，极端的情况下，两者的数据有可能重叠而导致浪费。\
比如，当你发起一个完全新的请求，那么数据库服务需要从磁盘读取数据块，这些块首先会进入操作系统缓存，然后拷贝到数据库缓存。虽然最终，这些缓存中的数据都会被刷出去，但在一段时间内，它在操作系统缓存和数据库缓存中都存在，因此设置 shared_buffers 一个恰当的值可以减少重复数据的数量。

### shared_buffers 的起点设置

设定一个好的 shared_buffers 值很难，通常我们都会说一个相对内存的比率。有部分人主张仅使用内存的 10-15%。论文[Tuning
Database Configuration Parameters with
iTuned](http://www.cs.duke.edu/~shivnath/papers/ituned.pdf)非常详尽的给出了一个有关该值的学术解释。这片文档提到他们在一台 1G 内存的机器上，发现设置 shared_buffers 设置为内存的 40%为其最佳值，并据此进行了非常广泛的查询测试。\
一般来说，如果你服务器上 OS 对内存开销很小，那么 shared_buffers 设置为内存的 25%是一个不错的起点，虽然不是最佳设置，但是一方面它可以有效减少重复的缓存数据，另外一方面，它也比缺省设置值要高很高，相比效率也有较大提升。

### 缓存查询检测

下面用一个实例来说明如何利用 pg_buffercache 来模块来检测缓存的使用
我们设置 shared_buffers
为 512M，然后使用 pgbench 创建一个比例为 50 的数据库，其目的是数据库大小超过 shared_buffers 的值，以免所有的查询数据都能缓存到内存里。\

```sql
pgbench -i -s 50 pgbench
psql -hlocalhost -x -c "select pg_size_pretty(pg_database_size('pgbench')) as db_size"

-[ RECORD 1 ]---
db_size | 754 MB

pgbench -h localhost -S -c 8 -t 25000 pgbench
starting vacuum...end.

transaction type: SELECT only
scaling factor: 50
query mode: simple
number of clients: 8
number of threads: 1
number of transactions per client: 25000
number of transactions actually processed: 200000/200000
tps = 540.590358 (including connections establishing)
tps = 540.624004 (excluding connections establishing)
```

#### 缓存里的最大表

我们可以通过下的 SQL 语句来查询在缓存中的前 10 个表是哪些

```sql
pgbench=# SELECT
  c.relname,
  count(*) AS buffers
FROM pg_class c
  INNER JOIN pg_buffercache b
    ON b.relfilenode=c.relfilenode
  INNER JOIN pg_database d
    ON (b.reldatabase=d.oid AND d.datname=current_database())
GROUP BY c.relname
ORDER BY 2 DESC
LIMIT 10;
             relname              | buffers
----------------------------------+---------
 pgbench_accounts                 |   51346
 pgbench_accounts_pkey            |   13713
 pg_depend_reference_index        |      11
 pg_depend                        |       9
 pg_rewrite                       |       5
 pg_extension                     |       5
 pg_depend_depender_index         |       5
 pg_description                   |       5
 pg_statistic                     |       4
 pg_statistic_relid_att_inh_index |       4
(10 rows)
```

除掉系统表以后，我们可以看到缓存的绝大部分都用在了 pgbench_accounts 表以及它的主键索引上，这也应该是我们期望的。

#### 缓存内容统计

下面的查询语句可以查看到，目前缓存里，数据量居前 10 位的对象是哪些，占用缓存比率是多少，以及相比对象大小的比率是多少

```sql
pgbench=#SELECT
  c.relname,
  pg_size_pretty(count(*) * 8192) as buffered,
  round(100.0 * count(*) /
    (SELECT setting FROM pg_settings
      WHERE name='shared_buffers')::integer,1)
    AS buffers_pct,
  round(100.0 * count(*) * 8192 /
    pg_relation_size(c.oid),1)
    AS pct_of_rel
FROM pg_class c
  INNER JOIN pg_buffercache b
    ON b.relfilenode = c.relfilenode
  INNER JOIN pg_database d
    ON (b.reldatabase = d.oid AND d.datname = current_database())
GROUP BY c.oid,c.relname
ORDER BY 3 DESC
LIMIT 10;
            relname              |  buffered  | buffers_pct | pct_of_rel
----------------------------------+------------+------------+-------------
 pgbench_accounts                 | 402 MB     |       78.5 |      62.8
 pgbench_accounts_pkey            | 107 MB     |       20.9 |     100.0
 pg_operator_oid_index            | 32 kB      |        0.0 |     100.0
 pg_opclass_oid_index             | 16 kB      |        0.0 |     100.0
 pg_namespace                     | 8192 bytes |        0.0 |     100.0
 pg_operator                      | 88 kB      |        0.0 |      78.6
 pg_amproc_fam_proc_index         | 24 kB      |        0.0 |      75.0
 pg_index                         | 24 kB      |        0.0 |     100.0
 pg_statistic_relid_att_inh_index | 32 kB      |        0.0 |     100.0
 pg_index_indrelid_index          | 16 kB      |        0.0 |     100.0
(10 rows)
```

注意，函数 pg_relation_size()不包含存储在 TOAST 表中的数据。如果你的 PostgreSQL 是 9.0 以上版本，可以使用 pg_table_size()函数来代替，它包含了 TOAST 表数据。

### 总结

一个数据库服务可能会遇到各种工作模式，所以很难有一个死规则来说根据硬件条件就可以配置出优化的参数。\
有些应用程序偏向把大部分内存专用于数据库，并从中获得好处，而有些应用程序则会因为这样的配置遇到 checkpoint 峰值问题。\
PostgreSQL
8.3 以后的版本提供了一些工具帮我们监控缓存这部分区域是如何使用的，再联合包括 OS 如何使用缓存等技术手段，我们就可以配置出一个靠谱的优化参数来。\
下面是一个针对数据库内存使用的规则：\

- 在服务启动时，PostgreSQL 分别一大块内存用户把数据库磁盘块导入内存，并进行读写操作

- 数据库缓存和操作系统缓存是协作关系而非替代关系，因此其大小应该是一个恰当的值而非一味的求大

- 默认情况下，分配给数据库的共享内存都很小，因此需要调整

- 崩溃后恢复是通过改变数据库文件之前将修改的内容先写入 WAL 来做到的

- Checkpoint 需要小心调整，既要限制崩溃后恢复时间又要考虑不影响系统的性能

- 系统视图 pg_stat_bgwriter 跟踪所有缓存的刷入刷出情况

- pg_buffercache
  模块用来查看缓存内有哪些内容，并据此判断缓存是太大还是太小

## 服务器配置调优

这一章主要围绕\$PGDATA/postgresql.conf
文件里的配置参数展开，基本上算是对<http://www.postgresql.org/docs/current/static/runtime-config.html>
一文的拷贝，不过我们重点关注一些重要参数的配置指导大纲，而不是介绍每个参数的含义。

另外一个有关调优的在线资源是官方 wiki 上的一篇文章:[Tuning Your PostgreSQL
Server](http://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)，这篇文章一直在更新以保持和当前主流版本的配置参数一致。

### 修改参数

PostgreSQL 参数可以有几种修改方式，有的可以在运行时直接修改，有的参数修改后，需要发出 SIGHUP 信号以便重新加载 postgresql.conf 文件来来生效，
有的修改后则需要重启数据库服务才生效，还有一些根本就不能修改。

如果确定这些参数属于哪种类型呢，我们可以直接查询数据库的字典来获得相关信息，就像下面这样：

```sql
pgbench=# select name,context from pg_settings;
              name               |  context
---------------------------------+------------
 application_name                | user
 archive_command                 | sighup
 archive_mode                    | postmaster
 archive_timeout                 | sighup
 block_size                      | internal
 dynamic_library_path            | superuser
```

context 字段所表示的具体含义，在官方文档里并没有详细给出，这里我们按照从难到易来介绍这些值的意思:\

internal

: 这种类型的参数都是编译时设置好的，你可以查看这些参数的值，但是除非重新编译，否则无法修改.

postmaster

: 这种类型的参数只有完全重启数据库服务才能生效，所有的共享内存设置参数都属于这类.

sighup

: 给服务发送 HUP 信号，使得服务可以重新加载 postgresql.conf 文件，从而使得这种类型的参数生效.

backend

: 这种类型的参数和 sighup 详细，只是它不对已经连接到服务器上的会话生效，只有新的会话才会启用修改后的参数，
属于这类的参数较少，log_connections 属于这类

superuser

: 顾名思义，这类参数只有数据库超级用户（默认一般是创建 postgres 数据库的用户）才能修改，这类参数大部分修改后即生效.

user

: 每个会话都可以修改的参数，且不互相影响，大部分的参数都属于这这种类型。

就我目前使用的 PostgreSQL 9.3 版本来说，各种类型的参数统计结果如下：

```sql
pgbench=# select context,count(context) as count
from pg_settings group by context;
  context   | count
------------+-------
 backend    |     5
 user       |    91
 internal   |    14
 postmaster |    37
 superuser  |    25
 sighup     |    52
(6 rows)
```

有几种办法可以让数据库重新加载 postgresql.conf 文件，一种是使用数据库自带的函数，就像下面的例子：

```sql
pgbench=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
LOG:  received SIGHUP, reloading configuration files
(1 row)
```

一种是直接给数据库进程发送 HUP 信号

```shell
lancy-imac:~ wgzhao$ ps -ef |grep /usr/bin/postgres
  501  2827     1   0 11:08上午 ttys000    0:00.61 /usr/bin/postgres
lancy-imac:~ wgzhao$ kill -HUP 2827
```

从命令行来看不会有任何输出，不过查看你的日志，或者终端，应该有下面类似的记录：\
`LOG:  received SIGHUP, reloading configuration files`

最后一种是通过 pg_ctl 指令来发送 SIGHUP 信号

```shell
pg_ctl reload
server signaled
```

### 服务端设置

\

listen_addresses

: 默认是仅允许本地链接，一般通过设置为'\*'来允许任何链接

max_connections

: 一般设置为 100，每进来一个链接，都需要占用一部分共享内存，因此其最大值和 shared_buffers 关系密切，
而且也有其他参数有关系，比如 work_mem

\

shared_buffers

: 这个参数在最后一章详细讲解，这是一个相当重要的参数

FSM

: 这里包括两个参数 max_fsm_pages 和 max_fsm_relations，在后面的章节会详细描述

\

log_line_prefix

: 默认设置为空，一个较好的建议值如下：\
 `log_line_prefix='%t:%r:%u@%d:[%p]: '`\
 其参数解释如下：

    %t

    :   时间戳

    %u

    :   数据库用户名

    %r

    :   链接的远程主机名

    %d

    :   链接的数据库名

    %p

    :   链接的进程ID

log_statement

: 有下面四个可选值来设置：\

    none

    :   不记录任何语句级信息

    ddl

    :   仅记录数据定义语言(Data Definition Language a.k.a
        DDL),比如create,drop，在生产系统上也可以使用
        这个参数，他记录的信息很少

    mod

    :   记录所有对值有修改的语句，基本上把除select语句以外的语句都记录了，量比较大

    all

    :   记录所有的语句，有相当大的开销，建议仅在调试时使用

log_min_duration_statement

: 记录那些执行时间超过该参数值的语句，单位是 ms，如果是 8.4 以后的版本，建议优先使用[auto_explain
模块](http://www.postgresql.org/docs/8.4/static/auto-explain.html)

### 专用数据库服务器参数设置

对于一台新的专用于数据库服务的设备，我们建议首先调整以下参数，然后运行一段时间，再根据实际情况调整：

1.  调整缺省的日志参数，以记录稍多一点信息

2.  shared_buffers 设置为物理内存的 25%，往上调整需要考虑到 checkpoint 的额外开销

3.  评估最大连接数，这是一个硬性限制，一旦超过最大连接数，则会被拒绝

4.  用上述参数启动服务，注意看还有多少内存可用于文件系统缓存

5.  调整 effective_cache_size 为 shared_buffers + OS cache

6.  OS cache / max_connections / 2
    其结果设置为 work_mem 的上限，如果你的应用程序不依赖于排序性能，可设置得更小

7.  maintenance_work_mem 设置为每 GB 内存 50MB

8.  checkpoint_segments 不小于 10，如果你服务器硬件具有 BBWC(Battery-backed
    write cache)功能，设置为 32 会更好

9.  如果你的系统对 wal_sync_method 的缺省设置方式不安全，更换安全的参数，一般都不需要修改

10. 修改 wal_bufffers 为 16MB

### 总结

PostgreSQL 里可以调整的参数差不多有 200 个，针对你的应用程序，想把这些参数都设置为最佳几乎是一件不可能的任务，这里给出的指南
也只是一个通常的考虑，以避免常见的瓶颈。

- 服务器的缺省配置参数使用非常少的内存以及记录很少的信息，这两点是必须要调整的

- 与内存有关的调整主要是 shared_buffers 和 work_mem，仔细调整，主要不要出现 OOM(Out
  of memory)现象

- 查询计划需要知道内存状态，有好的表信息统计有助于查询计划做出正确的判断

- autovacuum 进程很关键，它确保查询计划能获得正确的信息

- 大部分情况下，参数的调整都不需要重启服务，很多参数修改后直接生效

## 例行维护

## 数据库性能测试

这一章主要讨论 PostgreSQL 自带的 pgbench 的使用

### pgbench 缺省测试

pgbench 最开始是面向 TPC 测试的工具，主要测试[TPC-B](http://www.tpc.org/tpcb/)，1990 年开始开发，其模型是一个简单的银行
应用程序，包括支行、柜员和账户等信息。\
其主要的表的定义如下：

```sql
CREATE TABLE pgbench_branches(bid int not null, bbalance int, filler
char(88));
ALTER TABLE pgbench_branches add primary key (bid);
CREATE TABLE pgbench_tellers(tid int not null,bid int, tbalance
int,filler char(84));
ALTER TABLE pgbench_tellers add primary key (tid);
CREATE TABLE pgbench_accounts(aid int not null,bid int, abalance
int,filler char(84));
ALTER TABLE pgbench_accounts add primary key (aid);
CREATE TABLE pgbench_history(tid int,bid int,aid int,delta int, mtime
timestamp,filler char(22));
```

### 规模检测

模型中，支行的数量被称为数据库规模，每个支行包括 10 个柜员和 100,000 个账户，初始化数据库时，可以通过执行参数来设定数据库的规模，比如:

        $ createdb pgbench
        $ pgbench -i -s 10 pgbench

上述指令表示用 10 个支行的数据库规模填充数据库 pgbench，运行测试时，如果执行-s 的参数，则 pgbench 采纳这个参数，如果没有提供，则通过下面的 SQL 指令猜测数据库规模:
/select count(\*) from pgbench_branches;/

### 查询脚本定义

pgbench 内建的标准测试所需要的脚本和 SQL 语句直接编译进了程序中，当然我们也可以定制自己需要测试的脚本，其详细情况，可以参考[对应的文档](http://www.postgresql.org/docs/current/static/pgbench.html)\
缺省的交易脚本，我们称之为"TPC-B"交易，大致类似下面的语句组成:

```text
\set nbranches :scale
\set ntellers 10 * :scale
\set naccounts 100000 * :scale
\setrandom aid 1 :naccounts
\setrandom bid 1 :nbranches
\setrandom tid 1 :ntellers
\setrandom delta -5000 5000
BEGIN;
UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
END;
```

脚本的前 3 行依据:scale 变量计算有多少支行、柜员以及账户。接下来 4 行创建随机数，模拟银行交易。脚本的核心代码是由 5 个语句组成的事务，每个语句对数据库的影响都不同:\

update pgbench_accounts

: 作为数据库中最大的表，它最有可能引发磁盘 I/0 瓶颈

select abalance

: 因为之前的 update 语句已经把该语句需要查询的信息都缓存起来了，因此它增加的开销很小

update pgbench_tellers

: 相比 accounts 表，这个表小很多，能够在缓存到内存，因此它可能对引发锁的问题

update pgbench_branches

: 这个表相当小，整个内容都可以被缓存，所以访问速度不是问题，但是正因为小，锁有可能称为其访问瓶颈

insert into pgbench_history

: 历史表作为只追加表，表上无索引

这是缺省测试，还有两个别的参数可以稍微改变测试行为:

-N

: 和上面的测试相同，只是跳过两个小表(tellers,branches)的更新参数，这降级的锁的潜在问题

-S

: 只对 accounts 表做查询操作，因此它不需要把语句组成一个事务

仅做查询操作而跳过所有的更新操作对于检查缓存大小，测试最大 CPU 速度很有帮助。

#### 为 pgbench 配置数据库

内建的测试行为，其他语句都很简单：绝大部分是通过主键索引查询，没有表链接，也没用负责查询类型。\
但这些语句都会产生很重的写压力，需要缓存能跟得上\
基于此，我们可以把可以调整的参数分为三个主要类型:

- 重点关注和调整的参数:
  shared_buffers,checkpoint_segments,autovacuum,wal_buffers,checkpoint_completion_target

- 显著影响测试结果的参数:
  wal_sync_method,synchronouse_commit,wal_writer_delay

- 无关的:
  effective_cache_size,default_statistics_target,work_mem,random_page_cost 等

## 数据库索引

## 查询优化

### 填充样例数据

为了演示查询优化的过程，我们需要填充一些样例数据，PostgreSQL 的 wiki 上给出了一些常用的样例数据列表，可以参考这个链接：[http://wiki.postgresql.org/wiki/
Sample_Databases](http://wiki.postgresql.org/wiki/ Sample_Databases)
这里选取 Dell Store 2 这个有 Dell 贡献的样例，安装步骤如下：

```shell
wget http://pgfoundry.org/frs/download.php/543/dellstore2-normal-1.0.tar.gz
tar xvfz dellstore2-normal-1.0.tar.gz
cd dellstore2-normal-1.0/
createdb dellstore2
createlang plpgsql dellstore2
psql -f dellstore2-normal-1.0.sql -d dellstore2
psql -d dellstore2 -c "VACUUM VERBOSE ANALYZE"
```

该样例数据库大概 23MB

```sql
psql -hlocalhost -d dellstore2 -c "select pg_size_pretty(pg_database_size('dellstore2'))"
 pg_size_pretty
----------------
 23 MB
(1 row)
```

其主要的表和索引如下：

```sql
dellstore2=# \dt+
        List of relations
 Schema |    Name    |  Size   |
--------+------------+---------+
 public | categories | 40 kB   |
 public | cust_hist  | 2648 kB |
 public | customers  | 3944 kB |
 public | inventory  | 472 kB  |
 public | orderlines | 3112 kB |
 public | orders     | 832 kB  |
 public | products   | 840 kB  |
 public | reorder    | 0 bytes |
(8 rows)


dellstore2=# \di+
            List of relations
 Schema |          Name          |  Size   |
--------+------------------------+---------+
 public | categories_pkey        | 16 kB   |
 public | customers_pkey         | 456 kB  |
 public | inventory_pkey         | 240 kB  |
 public | ix_cust_hist_customerid| 1336 kB |
 public | ix_cust_username       | 624 kB  |
 public | ix_order_custid        | 280 kB  |
 public | ix_orderlines_orderid  | 1336 kB |
 public | ix_prod_category       | 240 kB  |
 public | ix_prod_special        | 240 kB  |
 public | orders_pkey            | 280 kB  |
 public | products_pkey          | 240 kB  |
(11 rows)
```

## 数据库活动和统计

PostgreSQL 有一个统计收集的子系统，它可以监控数据库内部各方面的信息。服务器的每个处理会发送统计消息给收集器，最后统计器汇总消息并发布。
缺省情况下，统计器没半秒(编译时指定的值)更新统计并发布，每一个统计值可以通过一套函数来返回，详细情况可以参考这个文档：<http://www.postgresql.org/docs/current/static/monitoring-stats.html>

### 统计视图

通过组织一系列的视图，以方便人们能够快速访问统计收集器的输出，阅读有关视图的源代码对于理解其是如何工作的非常有帮助，如果你有 PostgreSQL 的源代码，那么可以阅读
src/backend/catalog/system_views.sql 文件。如果你已经安装了数据库，可以阅读 share/system_views.sql，或者你也可以直接下载其源代码:<http://anoncvs.postgresql.org/cvsweb.cgi/pgsql/src/backend/catalog/system_views.sql>
我们可以看一个简单的视图来理解它一般是如何组织的，下面是 8.3 版本中 pg_stat_bgwriter 视图源代码：

```sql
CREATE VIEW pg_stat_bgwriter AS
SELECT
pg_stat_get_bgwriter_timed_checkpoints() AS checkpoints_timed,
pg_stat_get_bgwriter_requested_checkpoints() AS checkpoints_req,
pg_stat_get_bgwriter_buf_written_checkpoints() AS buffers_checkpoint,
pg_stat_get_bgwriter_buf_written_clean() AS buffers_clean,
pg_stat_get_bgwriter_maxwritten_clean() AS maxwritten_clean,
pg_stat_get_buf_written_backend() AS buffers_backend,
pg_stat_get_buf_alloc() AS buffers_alloc;
```

这个视图其实只是把各种独立的统计函数组织在一起，从而输出一个完整的信息，其他统计视图结构上也和这类似。

### 累积视图和实时视图

有两种类型的信息可以从数据库获得监控信息。主要的统计信息保存在计数器里。当你创建一个新的数据库时，计数器设置为 0，随着和统计相关的活动增加而增加。
计数器包括 pg_stat_database,pg_stat_bgwriter 以及所有命名以 pg_stat 开头的视图。\
有确切的方法把计数器清零，不过 PostgreSQL 版本不同，方法不同：

Version 8.1

: pg_stat_reset()重置所有统计，在 postgresql.conf 设置 stats_reset_on_server_start=on 会使得每次数据库服务重启都充值统计

Version 8.2

: pg_stat_reset()仅重置块以及行级别统计，而 stats_reset_on_server_start=on 则重置所有统计,包括数据库端和集群端。

Version 8.3,8.4

: pg_stat_reset()只重置当前数据库的所有统计，没法办法可以重置集群段的统计，比如 pg_stat_bgwriter。

Version 9.0

: pg_stat_reset()仅重置当前数据库的所有统计。pg_stat_reset_shared('bgwriter')可以重置 pg_stat_bgwriter.
pg_stat_reset_single_table_counters() ，
pg_stat_reset_single_function_counters()可以重置单独的表、索引以及函数统计

知道统计视图里哪些是累加的，哪些是固定的（比如表名）很重要。为了搞清楚累加的字段，你可以用带时间戳的方式定期抓取数据，然后观察这些数据在这个时间段内是如何变化的，这章的最后会给出人工抓取 pg_stat_bgwriter 数据。\
除了这些直观的统计计数器外，还有一些数据库活动的实时视图，它给出了数据库此刻所发生的活动的快照。这些视图包括 pg_stat_activity,pg_locks,pg_prepared_xacts.

### 表统计

每个表的基本统计视图为 pg_stat_all_tables，这个视图为了方便有分成了两个，一个是和系统表有关的 pg_stat_sys_tables，一个是和用户表 pg_stat_user_tables。后者是我们重点关注的。\
视图的第一个用处是我们可以使用这些数据来监控 vacuum 在你数据库里运作得如何。\
我们可以使用这个视图来判断这些表是否通过顺序扫描或者索引扫描访问过

```sql
pgbench=# select schemaname,relname,seq_scan,idx_scan,
          cast(idx_scan as numeric) / (idx_scan + seq_scan)
          as idx_scan_pct from pg_stat_user_tables
          where (idx_scan + seq_scan) > 0 order by idx_scan_pct;

 schemaname |     relname      | seq_scan | idx_scan | idx_scan_pct
 -----------+------------------+----------+----------+---------------
 public     | pgbench_tellers  |        1 |        0 | 0.000000
 public     | pgbench_branches |        4 |        0 | 0.000000
 public     | pgbench_accounts |        1 |   600000 | 0.999998
(3 rows)
```

在极小的 pgbench 数据库里，访问 pgbench_branches 和 pgbench_tellers 是通过顺序扫描访问的，因为他们都很小，可以载入到一个数据页内。而 pgbench_accounts 就很大了，因此 select 语句访问时，需要通过索引来查询数据。\
我们对表更感兴趣的是确切的知道这个表被这些扫面所处理过的元组,下面的视图查询可以获得这些信息：

```sql
pgbench=# select relname,seq_tup_read,idx_tup_fetch,
          cast(idx_tup_fetch as numeric) / (idx_tup_fetch + seq_tup_read)
          as idx_tup_pct
          from pg_stat_user_tables
          where (idx_tup_fetch +seq_tup_read) > 0 order by idx_tup_pct;

     relname      | seq_tup_read | idx_tup_fetch | idx_tup_pct
------------------+--------------+---------------+--------------
 pgbench_tellers  |          500 |             0 | 0.000000
 pgbench_branches |          200 |             0 | 0.000000
 pgbench_accounts |      5000000 |        600000 | 0.107143
(3 rows)
```

下面这个例子用来查询热(HOT)数据多长时间被用来更新:

```sql
pgbench=# select relname,n_tup_upd,n_tup_hot_upd,
          cast(n_tup_hot_upd as numeric) / n_tup_upd as hot_pct
          from pg_stat_user_tables
          where n_tup_upd > 0 order by hot_pct;

     relname      | n_tup_upd | n_tup_hot_upd | hot_pct
------------------+-----------+---------------+-----------
 pgbench_accounts |     24681 |          3431 | 0.139014
 pgbench_tellers  |     24681 |         24428 | 0.989749
 pgbench_branches |     24680 |         24680 | 1.000000
(3 rows)
```

从输出来看，pgbench_tellers 和 pgbench_branches 的数据更新几乎全部在热数据中完成，这有着很高的效率，但是对于大表 pgbench_accounts 而言，热数据的更新相比所有数据的更新，比率很低，从而性能不高，因此需要调整。\
最后一个好处是可以通过视图来思考在表上插入/更新/删除的特征:

```sql
pgbench=# SELECT relname,
(cast(n_tup_ins AS numeric)/(n_tup_ins + n_tup_upd + n_tup_del))::numeric(6,5) AS ins_pct,
(cast(n_tup_upd AS numeric)/(n_tup_ins + n_tup_upd + n_tup_del))::numeric(6,5) AS upd_pct,
(cast(n_tup_del AS numeric)/(n_tup_ins+ n_tup_upd + n_tup_del))::numeric(6,5) AS del_pct
FROM pg_stat_user_tables ORDER BY relname;

     relname      | ins_pct | upd_pct | del_pct
------------------+---------+---------+---------
 pgbench_accounts | 0.97916 | 0.02084 | 0.00000
 pgbench_branches | 0.00052 | 0.99948 | 0.00000
 pgbench_history  | 1.00000 | 0.00000 | 0.00000
 pgbench_tellers  | 0.00516 | 0.99484 | 0.00000
(4 rows)
```

这个输出证实了 pgbench_history 是我们之前称之为的只附加表，在这个表上只有插入，永远都不会有更新和删除。它也暗示了 pgbench_branches 和 pgbench_tellers 有大量的更新操作，而插入操作微乎其微，删除操作则根本就没有。
这些信息对于我们判断表是否需要周期性的执行 reindex 操作很有帮助。
同样的，对于删除操作的比率观察，也可以让我们思考如果这个比较较高，则可以执行诸如 CLUSTER 操作来清理空间。

### 表 I/O

pg_statio_all_tables(pg_statio_user_tables/pg_statio_sys_tables)用来集中统计数据库上的物理 I/O。
第一对要监控的字段是用 shared_buffers 来缓存表数据的结构，在这个上下文里称为堆(heap)，当发生一个读操作时，数据判断是读取数据库缓存(heap
block hit)块还是要求执行操作系统读(heap block read)才能满足要求。

```sql
pgbench=# select relname,
(cast(heap_blks_hit as numeric)/(heap_blks_hit + heap_blks_read))::numeric(6,5) as hit_pct,
heap_blks_hit,heap_blks_read
from pg_statio_user_tables
where (heap_blks_hit + heap_blks_read) > 0 order by hit_pct;

     relname      | hit_pct | heap_blks_hit | heap_blks_read
------------------+---------+---------------+----------------
 pgbench_accounts | 0.59564 |        906335 |         615280
 pgbench_history  | 0.99316 |        109787 |            756
 pgbench_tellers  | 0.99977 |        197282 |             46
 pgbench_branches | 0.99988 |        280172 |             34
(4 rows)
```

要注意的是，并不是所有的 heap blocks
read 都会转化为物理磁盘 I/O，他们有可能是从操作系统缓存读取。当前，数据库没有办法知道到底是哪种读取方式。
不过我们可以通过类似\"[Dtrace](http://en.wikipedia.org/wiki/DTrace)\"这样的工具来跟踪读操作，而从做出正确的判断。\
查询表上每个索引所用到的磁盘 I/O 语句如下：

```sql
pgbench=# select relname,
(cast(idx_blks_hit as numeric)/(idx_blks_hit + idx_blks_read))::numeric(6,5) as hit_pct,
idx_blks_hit,idx_blks_read
from pg_statio_user_tables
where (idx_blks_hit + idx_blks_read) > 0 order by hit_pct;

     relname      | hit_pct | idx_blks_hit | idx_blks_read
------------------+---------+--------------+---------------
 pgbench_branches | 0.91667 |           77 |             7
 pgbench_accounts | 0.97414 |      2555866 |         67839
 pgbench_tellers  | 0.99990 |       194106 |            19
(3 rows)
```

这里看不到 pgbench_history 表的信息，因为该表没有索引。\
表 I/O 统计也可以包括 TOAST 表的统计，查询方式如下：

```sql
pgbench=#select relname,
(heap_blks_read + toast_blks_read + tidx_blks_read) as total_blks_read,
(heap_blks_hit + toast_blks_hit + tidx_blks_hit) as total_blks_hit
from pg_statio_user_tables;
```

### 索引统计

pg_stat_all_indexes(pg_stat_user_indexes/pg_stat_sys_indexes)视图用于统计索引信息，它给出的包括包括有多少索引扫描被处理以及通过索引扫描返回了多少行数据。\
这个视图的字段名字很复杂，很难区分。比如下面两个字段名字差不多，差别也很细微：

idx_tup_read

: 通过索引扫描返回的索引条目数

idx_tup_fetch

: 通过该索引执行简单索引扫描后获取的表记录数。这个数通常比 idx_tup_read 小一点，因为它不包括死的或者还没有提交的行。
另外，这里的"简单"意思是说索引访问不是位图索引扫描(bitmap index
scan)

因为有太多的场景是 idx_tup_fetch 不记录的，因此如果你想追踪一个索引到底被系统使用的频率度有多少，idx_tup_read 会是更好的选择。\
通过这个视图，我们可以知道该索引扫墓所返回的行记录平均数以及索引是如何被使用的:

```sql
pgbench=# select indexrelname,
(cast(idx_tup_read as numeric) / idx_scan)::numeric(6,5) as avg_tuples,
idx_scan,idx_tup_read
from pg_stat_user_indexes
where idx_scan > 0 ;

     indexrelname      | avg_tuples | idx_scan | idx_tup_read
-----------------------+------------+----------+--------------
 pgbench_tellers_pkey  |    1.00280 |    96437 |        96707
 pgbench_accounts_pkey |    1.07489 |   812874 |       873753
(2 rows)
```

输出显示基本上每次索引扫面返回一条记录，这对于主键索引而言并不奇怪。avg_tuples 比 1.0 稍大一点点，反映出索引扫描偶有扫描到无效行。\
pg_stat_user_indexes 所统计的信息最大的用处是用来判断一个所有是否被你的应用真正使用。因为索引会增加系统的而外开销，因此如果一个索引并不是被使用，那就应该删除。
查找这样的索引的最简单办法就是查询 idx_scan 值很小的所以你，比如像下面这样：

```sql
pgbench=# select schemaname,relname,indexrelname,idx_scan,
          pg_size_pretty(pg_relation_size(i.indexrelid)) as idx_size
          from pg_stat_user_indexes i join pg_index using (indexrelid)
          where indisunique is false order by idx_scan,relname;

 schemaname | relname | indexrelname | idx_scan | idx_size
------------+---------+--------------+----------+-----------
(0 rows)
```

这里特意排除了强制唯一约束，因为这些索引即便很少使用，也不应该删除。

### 索引 I/O

和表 I/O 视图类似，也是分为三个视图,用法也差不多

```sql
pgbench=# select indexrelname,
(cast(idx_blks_hit as numeric)/(idx_blks_hit + idx_blks_read))::numeric(6,5) as hit_pct,
idx_blks_hit,idx_blks_read from pg_statio_user_indexes
where (idx_blks_hit + idx_blks_read) >0 order by hit_pct;

     indexrelname      | hit_pct | idx_blks_hit | idx_blks_read
-----------------------+---------+--------------+---------------
 pgbench_branches_pkey | 0.91667 |           77 |             7
 pgbench_accounts_pkey | 0.97414 |      2555866 |         67839
 pgbench_tellers_pkey  | 0.99990 |       194106 |            19
(3 rows)
```

### 数据库统计

视图 pg_stat_database 可以帮助我们获取有关数据库级别的统计信息：

```sql
pgbench=# select datname,blks_read,blks_hit,tup_returned,
          tup_fetched,tup_inserted as tup_ins,
          tup_updated as tup_upd,tup_deleted as tup_del
          from pg_stat_database
          where datname = 'pgbench';

 datname | blks_read | blks_hit | tup_returned | tup_fetched | tup_ins | tup_upd | tup_del
---------+-----------+----------+--------------+-------------+---------+---------+-------------
 pgbench |    689749 | 11227681 |     19441670 |     1515691 | 5319222 |  635663 |  100
(1 row)
```

除此之外，试图还有一些有用的事务提交统计信息。

```sql
pgbench=# select datname,numbackends,xact_commit,xact_rollback
          from pg_stat_database;
  datname   | numbackends | xact_commit | xact_rollback
------------+-------------+-------------+---------------
 template1  |           0 |       26616 |             2
 template0  |           0 |           0 |             0
 postgres   |           0 |       27441 |           337
 wgzhao     |           0 |       31578 |           302
 test       |           0 |       27711 |            45
 pgbench    |           1 |      840155 |            25
 db1        |           0 |           0 |             0
 db2        |           0 |           0 |             0
 dellstore2 |           0 |        1317 |             1
(9 rows)
```

### 连接和活动

pg_stat_activity 提供了每一个客户端当前在服务器上的活动快照。

我们可以使用这个视图来查看一个后台进程运行了多久已经此刻是否正在等待什么:

```sql
pgbench=# select pid,waiting,
          current_timestamp - least(query_start,xact_start) as runtime,
          substr(query,1,25) as current_query
          from pg_stat_activity
          where not pid = pg_backend_pid();

  pid  | waiting |     runtime     |       current_query
-------+---------+-----------------+---------------------------
 11987 | f       | 00:00:00.005063 | END;
 11989 | f       | 00:00:00.00341  | UPDATE pgbench_tellers SE
 11988 | f       | 00:00:00.017561 | END;
 11991 | f       | 00:00:00.002855 | UPDATE pgbench_tellers SE
 11990 | f       | 00:00:00.002629 | UPDATE pgbench_accounts S
 11992 | f       | 00:00:00.002998 | UPDATE pgbench_tellers SE
 11993 | f       | 00:00:00.002823 | UPDATE pgbench_accounts S
 11994 | f       | 00:00:00.003072 | UPDATE pgbench_branches S
 11995 | f       | 00:00:00.002469 | UPDATE pgbench_tellers SE
 11996 | f       | 00:00:00.002852 | UPDATE pgbench_tellers SE
(10 rows)
```

### 磁盘使用

获得一个表或者索引使用了多少磁盘空间的最基本方法是运行 pg_relation_size()函数，它经常和 pg_size_pretty()函数联合，以便输出方便阅读的大小。

比如下的例子就是查询当前数据库下所有的表所使用的磁盘大小：

```sql
pgbench=# select relname,
       pg_size_pretty(pg_relation_size(C.oid)) as "size"
       from pg_class C
       left join pg_namespace N on (N.oid = C.relnamespace)
       where relnamespace not in
       (select oid from pg_namespace where nspname = 'pg_catalog'
            or nspname = 'informantion_schema')
       order by pg_relation_size(C.oid) desc limit 10;

        relname        |  size
-----------------------+---------
 pgbench_accounts      | 650 MB
 pgbench_accounts_pkey | 107 MB
 pgbench_history       | 3088 kB
 pg_toast_2618         | 320 kB
 sql_features          | 56 kB
 pgbench_tellers       | 40 kB
 pgbench_tellers_pkey  | 40 kB
 pgbench_branches      | 24 kB
 pg_toast_2619         | 16 kB
 pgbench_branches_pkey | 16 kB
(10 rows)
```

这个查询语句的主要问题是表大小里没有包含 TOAST 表的大笑，虽然可以使用 pg_total_relation_size()，但是她又包含了索引大小。

PostgreSQL
9.0 以后，有一个名为 pg_table_size()的函数包括了除索引外的所有其他表信息。而 pg_indexes_size()则给出了一个表的所有索引大小。
这些函数所计算的大小关系如下：

        pg_total_relation_size = pg_table_size + pg_indexes_size
        pg_table_size = pg_relation_size + toast table + toast indexes + FSM

下面的查询显示了 TOAST 和索引大小:

```sql
pgbench=# select relname,relkind as "type",pg_size_pretty(pg_table_size(C.oid)) as size,
          pg_size_pretty(pg_indexes_size(C.oid)) as idxsize,
          pg_size_pretty(pg_total_relation_size(C.oid)) as "total"
          from pg_class C left join pg_namespace N on (N.oid= C.relnamespace)
          where C.relnamespace not in (11,11728,99) and relkind in ('r','i')
          order by pg_total_relation_size(C.oid) desc limit 20;

        relname        | type |  size   | idxsize |  total
-----------------------+------+---------+---------+---------
 pgbench_accounts      | r    | 650 MB  | 107 MB  | 757 MB
 pgbench_accounts_pkey | i    | 107 MB  | 0 bytes | 107 MB
 pgbench_history       | r    | 3112 kB | 0 bytes | 3112 kB
 pgbench_tellers       | r    | 72 kB   | 40 kB   | 112 kB
 pgbench_branches      | r    | 56 kB   | 16 kB   | 72 kB
 pgbench_tellers_pkey  | i    | 40 kB   | 0 bytes | 40 kB
 pgbench_branches_pkey | i    | 16 kB   | 0 bytes | 16 kB
(7 rows)
```

### 缓存,后台写以及 checkpoint

pg_stat_bgwriter 可以让我们追踪每个数据页在缓存内外的流向，我们可以通过查询该视图回答下面的问题：

- 执行 checkpoint 的时间中，基于活动的请求占比多少，基于时间间隔的比例又是多少？

- 平均 checkpoint 写出多少数据？

- 写出的数据中，checkpoint 和后台写分别占比多少？

```sql
pgbench#SELECT
  (100 * checkpoints_req) / (checkpoints_timed + checkpoints_req)
    AS checkpoints_req_pct,
  pg_size_pretty(buffers_checkpoint * block_size / (checkpoints_timed +
       checkpoints_req))
    AS avg_checkpoint_write,
  pg_size_pretty(block_size * (buffers_checkpoint + buffers_clean +
       buffers_backend)) AS total_written,
  100 * buffers_checkpoint / (buffers_checkpoint + buffers_clean +
      buffers_backend) AS checkpoint_write_pct,
  100 * buffers_backend / (buffers_checkpoint + buffers_clean +
        buffers_backend) AS backend_write_pct,
  *
FROM pg_stat_bgwriter,(SELECT cast(current_setting('block_size') AS integer)
     AS block_size) AS bs;

-[ RECORD 1 ]---------+------------------------------
checkpoints_req_pct   | 1
avg_checkpoint_write  | 742 kB
total_written         | 4588 MB
checkpoint_write_pct  | 43
backend_write_pct     | 56
checkpoints_timed     | 2713
checkpoints_req       | 38
checkpoint_write_time | 2475096
checkpoint_sync_time  | 253253
buffers_checkpoint    | 255206
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 332040
buffers_backend_fsync | 0
buffers_alloc         | 503829
stats_reset           | 2012-08-14 17:00:19.529427+08
block_size            | 8192
```

从这个结果来看，只有 1%的 checkpoint 是因为达到了 WAL 达到了需要写出的要求，而绝大部分 checkpoint 都是因为 checkpoint_timeout 参数所设置的
间隔时间来临而执行的。

checkpoint 平均写出的数据是 742KB，相当小。

56%的数据写出是后台写进程，这暗示这系统的配置还不足够好，我们应该更多使用 checkpoint 来写出数据。发生这个情况的原因可能是 shared_buffers 配得小了点。

### 保存 pg_stat_bgwriter 快照

为了使上述统计数据更有用，我们可以定期保存快照，我们只需要创建一个表，然后把快照写入表即可，类似如下：

```sql
pgbench=# create table pg_stat_bgwriter_snapshot as
          select current_timestamp,* from pg_stat_bgwriter;
SELECT 1
pgbench=# insert into pg_stat_bgwriter_snapshot
         (select current_timestamp,* from pg_stat_bgwriter);
INSERT 0 1
```

我们应该至少保存一个小时的快照信息，一天更好。这样我们才有可能通过这些快照获得一些更有用的信息，比如：

```sql
SELECT
  cast(date_trunc('minute',start) AS timestamp) AS start,
  date_trunc('second',elapsed) AS elapsed,
  date_trunc('second',elapsed / (checkpoints_timed + checkpoints_req)) AS
    avg_checkpoint_interval,
  (100 * checkpoints_req) / (checkpoints_timed + checkpoints_req)
    AS checkpoints_req_pct,
  100 * buffers_checkpoint / (buffers_checkpoint + buffers_clean +
     buffers_backend) AS checkpoint_write_pct,
  100 * buffers_backend / (buffers_checkpoint + buffers_clean +
       buffers_backend) AS backend_write_pct,
  pg_size_pretty(buffers_checkpoint * block_size / (checkpoints_timed +
      checkpoints_req))
    AS avg_checkpoint_write,
  pg_size_pretty(cast(block_size * (buffers_checkpoint + buffers_clean +
      buffers_backend) / extract(epoch FROM elapsed) AS int8))
          AS written_per_sec,
  pg_size_pretty(cast(block_size * (buffers_alloc) / extract(epoch FROM
      elapsed) AS int8)) AS alloc_per_sec
FROM
( SELECT
  one.now AS start,
  two.now - one.now AS elapsed,
  two.checkpoints_timed - one.checkpoints_timed AS checkpoints_timed,
  two.checkpoints_req - one.checkpoints_req AS checkpoints_req,
  two.buffers_checkpoint - one.buffers_checkpoint AS buffers_checkpoint,
  two.buffers_clean - one.buffers_clean AS buffers_clean,
  two.maxwritten_clean - one.maxwritten_clean AS maxwritten_clean,
  two.buffers_backend - one.buffers_backend AS buffers_backend,
  two.buffers_alloc - one.buffers_alloc AS buffers_alloc,
  (SELECT cast(current_setting('block_size') AS integer)) AS block_size
  FROM pg_stat_bgwriter_snapshot one
  INNER JOIN pg_stat_bgwriter_snapshot two
    ON two.now > one.now
) bgwriter_diff
WHERE (checkpoints_timed + checkpoints_req) > 0;

start                   | 2010-04-09 19:52:00
elapsed                 | 00:17:54
avg_checkpoint_interval | 00:03:34
checkpoints_req_pct     | 80
checkpoint_write_pct    | 85
backend_write_pct       | 14
avg_checkpoint_write    | 17 MB
written_per_sec         | 94 kB
alloc_per_sec           | 13 kB
```

从输出来看，有 80%的 checkpoint 被调用，因为系统写满了 WAL,checkpoint 的调用平均间隔时间为 3.5 分钟，相当频繁。幸运的是，大部分缓存都被写出；
85%的缓存有 checkpoint 写出，这很好，因为 checkpoint 写算是最有效率的一种。每一个 checkpint 平均写出 17MB 数据。但是你计算所有的写操作，返现写到磁盘的速率
只有 94kB/s，这相当低。在读方面，缓存中 13kB/s 的速度被数据库分配用来满足查询要求。

对于一个持续 46 分钟的快照输出如下,这个阶段基本上处于空闲状态:

```text
start                   | 2010-04-09 20:10:00
elapsed                 | 00:46:26
avg_checkpoint_interval | 00:05:09
checkpoints_req_pct     | 0
checkpoint_write_pct    | 100
backend_write_pct       | 0
avg_checkpoint_write    | 910 bytes
written_per_sec         | 3 bytes
alloc_per_sec           | 0 bytes
```

注意到 checkpoint 的平均间隔是 5 分钟，而这恰好是 checkpoint_timeout 值，这说明这些 checkpoint 完全是由超时驱动。

这里的 alloc_per_sec 比率并不正好同等于从 OS 缓存的读取率，不过它暗示了内部缓存被重用的频率。

written_per_sec 值是把数据库表和索引写入到磁盘的平均值，它应该非常接近操作系统系统的平均写入磁盘的数值。

### 使用后台写统计调优

利用后台写统计信息来调整和 checkpoint 以及缓存的流程如下：

1.  定期收集 pg_stat_bgwriter 快照，这样就获得一个性能基线，此后，每次修改参数，重新产生快照，进行对比。依次循环，找到最优参数

2.  增加 checkpoint_segments 值直到几乎所有的 checkpoints 都是按超时驱动的，而不是按 activity 驱动。最终，90%以上应该基于超时
    而 avg_checkpoint_interval 应该接近 5 分钟（或者是你设置的 checkpoint_timeout 值）

3.  增加 shared_buffers 值在物理内存的 25%以上，而后参考快照，修改该参数

4.  当你的系统有效的使用大缓存时，checkpoint(不是 backends 或 background
    writer)执行时，缓存的百分比也应该增加，而总的 I/O
    written_per_sec 参数应该下降。
    如果最后是通过修改 shared_buffers 来达到这个标准，则需要返回步骤 3，重复这些步骤，知道性能曲线趋平。

5.  如果 shared_buffers 修改的值不是那么显著，则说明之前的值对目前的工作负荷足够了。

6.  现在你有了较好的 checkpoint_segements 和 shared_buffers 值。如果写出数据的大部分都来自 buffers_checkpoint，且 written_per_sec 显示的平均 I/O 看上去比较高（或者是执行写操作时，系统明显显得慢），
    应考虑增加 checkpoint_timeout 值，如果增加了，则需要重复上述流程。

## 监控和趋势分析

数据的性能直接和它底层的操作系统性有关联，而操作系统的性能又和底层的硬件有关联。因此只有这三者都处于工作良好的状态，才会有好的性能。

这就需要你有一个好的监控系统，用来获取所有层面的正确信息，然后做出正确的判断，并且可以预测什么系统系统会达到其负载极限，参数的改变是否对系统的提升长生了效果。

### unix 监控工具

这里主要介绍了 vmstat,iostat 工具的介绍和使用，这部分可以参考[Linux 下的一些 I/O 统计工具](http://blog.wgzhao.com/2012/08/22/some-way-to-io-statistics-on-linux/)一文

### 超负荷的系统例子

下面给出的例子中，数据库目录对应的磁盘布局是:

- pg_xlog: /dev/sdf1

- data: /dev/md0 Linux 软 RAID0，由 sdc1,sdd1 和 sde1 组成

为了加大系统的负载，我们还是用 pgbench 做测试，把 scale 设到 1000，客户端数设置到 64:

```shell
$ pgbench -i -s 1000 pgbench
$ pgbench -j 4 -c 64 -T 300 pgbench
```

在测试过程中，我们抓取 vmstat 一段时间的输出:

    procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
     r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
     2 45      0 128464   5608 2787264    0    0  1462 17739 1586 2984  4  4  0 93
     0 31      0 125356   5616 2789904    0    0  1232  1384 1591 3020  4  2  0 95
     0 44      0 121628   5616 2793660    0    0  1712  2152 1984 3953  4  3  0 93
     0 39      0 117164   5616 2797724    0    0  2144  1856 2270 4414  4  2  0 94
     0 43      0 112452   5616 2802036    0    0  2128  2224 2252 4514  4  3  0 93
     0 53      0 107376   5620 2806572    0    0  2576  2101 2060 4149  4  4  0 93
     0  1      0 119412   5520 2791864    0    0  1322 17266 2028 3780  4  3 13 81
     0  3      0 119048   5552 2789328    0    0    72  2426  801 1151  2  2 36 61
     0 26      0 103904   5628 2809296    0    0  1400  1396 2048 3909  5  3  0 93
     0 56      0 128208   5516 2785068    0    0  1808  1952 1546 3110  3  3 10 84
     0 59      0 124148   5516 2789132    0    0  2008  2240 2251 4595  3  2  0 95

从输出来看，cs 值的显著降低使得整个系统的吞吐也下来了。这里有点困惑的是，与此同时，wa 值也下来了也下来了，这是一个典型的场景，当系统负载太高时，它会因为
索资源竞争的原因而拒绝客户端的请求。我们从 cs 的最低点(1151)也能看出，其客户端进程非常少，仅有 1 个(对应行的 b 列显示)。这很有可能的原因是磁盘控制器的 cache 已经填满了，客户端都在等待
WAL 数据写回磁盘。

有时，系统慢有可能单纯是 I/O 吞吐原因，比如像下面的输出：

    # iostat –x 5
    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
               5.21    0.00    2.80   91.74    0.25    0.00
    Device rsec/s    wsec/s avgrq-sz avgqu-sz   await  svctm  %util
    sdc1  2625.60   2057.60    14.67    38.79  101.47   3.14 100.08
    sdd1  2614.40   2000.00    14.90    57.96  162.95   3.23 100.08
    sde1  2736.00   1963.20    14.64    48.50  135.26   3.12 100.00
    md0   7916.80   7206.40    10.60     0.00    0.00   0.00   0.00
    sdf1     0.00  22190.40    50.20     0.84    1.79   1.34  59.20

WAL 的写速大约 11MB/s。与此同时，数据库磁盘(md0)利用率达到 100%，然而读写速度只有 8MB/s(读+写)，这很可能此时数据库有很繁重随机 I/O，
而底层磁盘的随机 I/O 能力也就是 2MB/s 的样子。

这个数据库过程很有可能是 checkpoint 同步阶段，当大量的随机 I/O 写满磁盘缓存后，开始强制写入到磁盘。

我们还可以看下面更极端的例子，我们把 scale 设置到 4000

```shell
$ pgbench -i -s 4000 pgbench
$ pgbench -j 4 -c 64 -T 300 pgbench
```

执行期间，抓取一段时间的 vmstat 输出如下：

    # vmstat 1
    procs ----io---- --system--- -----cpu-------
    ￼r  b   bi    bo    in    cs    us sy id  wa st
    1  63  5444  9864   8576 18268  4   3  0   92  0
    0  42  4784  13916  7969 16716  4   2  3   90  0
    0  59  464   4704   1279  1695  0   0 25   75  0
    0  54  304   4736   1396  2147  0   0 25   74  0
    0  42  264   5000   1391  1796  0   0 10   90  0
    0  42  296   4756   1444  1996  1   0 10   89  0
    0  29  248   5112   1331  1720  0   0 25   75  0
    0  47  368   4696   1359  2113  0   0 23   76  0
    1  48  344   5352   1287  1875  0   0  0   100 0
    0  64  2692  12084  5604  9474  8   2  0   90  0
    1  63  5376  9944   8346 18003  4   3  20  74  0
    0  64  5404  10128  8890 18847  4   3  25  67  0

这里有 8 秒的时间间隔内，系统性能严重下降。cpu 耗时中，几乎没有 user
time。这符合所有客户端整等待某些资源的场景。这个场景出现的原因
在这儿很显然，wa 值很高，甚至到 100%，这说明服务器无法跟上磁盘 I/O 加载。

我们可以通过 iostat 的扩展选项查询此时磁盘负载在做些什么：

    # iostat –x 5
    avg-cpu:  %user   %nice %system %iowait  %steal   %idle
               2.35    0.00    1.85   85.49    0.15   10.16
    Device rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
    sdc1  2310.40  1169.60    14.63    39.85  147.59   4.21 100.00
    sdd1  2326.40  1216.00    14.53    71.41  264.60   4.10 100.00
    sde1  2438.40  1230.40    14.77    47.88  157.12   4.01  99.60
    md0   7044.80  4820.80    11.12     0.00    0.00   0.00   0.00
    sdf1     0.00 19040.00    70.31     4.20   15.31   2.10  56.88

输出显示读写速度很低，但是磁盘利用率达到了 100%，注意 await 指标，这么高的值表明磁盘等待时间过长，说明磁盘寻道时间占比很大，因而可以推断
随机读写占比很高。

### windows 监控工具

任务管理器是 Windows 系统下最简单和常用的系统监控工具，它提供了类似 UNIX
top 工具的输出视图。

如果需要功能强大一点的，可以从微软内部工具箱下载[进程浏览工具](http://technet.microsoft.com/en-us/sysinternals/bb896653)以及
[进程监控工具](http://technet.microsoft.com/en-us/sysinternals/bb896645)。相关用法可以参考工具自带的手册。

如果需要保存这些监控信息，类似 UNIX 的 sar 工具一样，微软也提供详细的指南，不同系统版本提供的方式不同，分列如下：

- Windows 2000,XP,Server 2003:
  <http://support.microsoft.com/kb/248345>

- Vista:
  <http://technet.microsoft.com/en-us/library/cc722173(WS.10).aspx>

- Windows 7,Windows Server 2008 R2:
  <http://technet.microsoft.com/en-us/library/dd744567(WS.10).aspx>

### 趋势分析软件

列举如下，具体的用法参考对应工具的问题：

- MRTG,RRDTool:
  这两个算是基础工具，用于数据的展现，可作为其他软件的附加组件

- Nagios:
  最常用的监控 PostgreSQL 的工具，内置了对 PostgreSQL 的监控插件[check_postgres](http://bucardo.org/wiki/Check_postgres)

- Cacti:
  也是非常流行的开源监控软件，它严重依赖 web 应用技术以及 RRDTool 工具，目前它内置的 PostgreSQL 监控工具还比较粗造，且还在开发中。详细的可以
  参考这个文档<http://wiki.postgresql.org/wiki/Cacti>

- Munin:
  相比 Nagios 和 Cacti，Munin 更小而且更新，它采用了和 Cacti 相同的 RRDTool 数据分析结构，它还可以集成到 Nagios 的报警结构中。针对 PostgreSQL 的监控插件
  是<http://muninpgplugins.projects.postgresql.org/>

- pgstatspack:
  这算是针对 PostgreSQL 的一个组件包，它的功能类似 pg_stat_bgwriter 试图--用于保存 PostgreSQL 定期的统计嘻嘻到数据库表里--以便以后分析。它的设计来源于
  针对 Oracle 的一个非常流程的工具包 statspack。可以通过[官方网站](http://pgstatspack.projects.postgresql.org/)了解更多，这里有一篇不错的[介绍文章](http://www.fuzzy.cz/en/articles/db-tuning-using- pgstatspack/)。

- Zenoss:
  Zenoss 相比上面提到的产品配置起来麻烦一点，它的核心是一套开源的监控解决方案，公司也提供了商业授权的企业版。它通过称为 Zenpack 的捆绑扩展组建来
  提供对服务提供监控和分析功能。最新发布的 PostgreSQL
  ZenPack 可以从下面的链接找到：<http://community.zenoss.org/docs/DOC-3389>

- Hyperic HQ:
  商业产品，集成了 PostgreSQL 监控,但支持的版本很低，目前仅支持到 8.3，具体可以参考这里<http://www.hyperic.com/products/postgresql-monitoring>

### 总结

## 连接池和缓存

### 连接池

连接池位于应用程序和数据库中间，连接到数据库的会话数固定，一般低于 100，运行期间，这些连接一直都保持。当应用程序有新的请求时，连接池的连接可以被重用。
PostgreSQL 里的 DISCARD
ALL 命令可以刷新一个新连接。当客户端断开时，连接池只是重置会话，并不实际从数据库断开。

连接池的数量在绝大部分场景下依赖于 CPU 的核数、数据库使用多少内存作为缓存以及底层磁盘的速度。

大部分用户认为最佳的连接池数量在 CPU 核数的 2 到 3 倍中间，如果你系统所带的磁盘很多，可以考虑比这个数大一点。

一般的一个迭代技术就是先把连接数设置为 100，然后根据系统的负载做调整。

#### pgpool-II

[pgpool](http://www.pgpool.net/mediawiki/index.php/Main_Page)是 PostgreSQL 上最老的连接池组件，目前还在持续开发中。pgpool-II 相比原来的版本在各方面都有了改进。

它的主要目的不仅仅是做连接池，它还可以提供负载均衡以及复制相关的功能。它甚至还支持某些并行查询设置。以下是它提供的一些功能的描述：

连接池

: pgpool-II 保持已经连接到 PostgreSQL 服务器的连接，
并在使用相同参数（例如：用户名，数据库，协议版本）
连接进来时重用它们。 它减少了连接开销，并增加了系统的总体吞吐量。

复制

: pgpool-II 可以管理多个 PostgreSQL 服务器。
激活复制功能并使在 2 台或者更多 PostgreSQL
节点中建立一个实时备份成为可能，
这样，如果其中一台节点失效，服务可以不被中断继续运行。

负载均衡

: 如果数据库进行了复制（可能运行在复制模式或者主备模式下），
则在任何一台服务器中执行一个 SELECT 查询将返回相同的结果。 pgpool-II
利用了复制的功能以降低每台 PostgreSQL 服务器的负载。 它通过分发
SELECT 查询到所有可用的服务器中，增强了系统的整体吞吐量。
在理想的情况下，读性能应该和 PostgreSQL 服务器的数量成正比。
负载均很功能在有大量用户同时执行很多只读查询的场景中工作的效果最好。

限制超过限度的连接

: PostgreSQL
会限制当前的最大连接数，当到达这个数量时，新的连接将被拒绝。
增加这个最大连接数会增加资源消耗并且对系统的全局性能有一定的负面影响。
pgpoo-II
也支持限制最大连接数，但它的做法是将连接放入队列，而不是立即返回一个错误。

并行查询

: 使用并行查询时，数据可以被分割到多台服务器上，
所以一个查询可以在多台服务器上同时执行，以减少总体执行时间。
并行查询在查询大规模数据的时候非常有效。

pgpool-II 使用 PostgreSQL
的前后台程序之间的协议，并且在前后台之间传递消息。
因此，一个（前端的）数据库应用程序认为 pgpool-II 就是实际的 PostgreSQL
数据库， 而后端的服务进程则认为 pgpool-II 是它的一个客户端。 因为
pgpool-II 对于服务器和客户端来说是透明的，
现有的数据库应用程序基本上可以不需要修改就可以使用 pgpool-II 了。

#### pgBouncer

[p](http://pgfoundry.org/projects/pgbouncer/)gBouncer 是 PostgreSQL 里能提高最高效率的连接池组件，该组件最初是 Skype 作为数据库扩展功能开发的。

pgBouncer 是单进程程序，针对每个连接并不长生新进程，其底层架构上依赖 UNIX 平台的 libevent 动态库，其内部的队列管理可以配置，以避免出现超时现象。

pgBouncer 会创建 pgbouncer 数据库，可以使用标准的 pgsql 工具连接到该库，也可以使用 SHOW 命令来获得有关连接池的内部信息。其终端接口接受像 PAUSE，RESUME 这样的命令来控制连接池。

pgBouncer 的另外一个特性是它支持对多个数据库服务器的连接。这就使得你的系统负载有可能分散到多个数据库服务器节点上。

如果你的有几百或者上千的连接数，他们超过了你服务器上 CPU 的个数，那么 pgBouncer 是你的第一考虑。

#### 应用服务器连接池

有的应用程序可能并不需要数据库层面的连接池，它偏向于使用自己的连接池，大部分流行的 Java 应用服务，包括 Tomcat,JBoss 等，都有自己的连接池组件。
比如 Tomcat 就使用称为 DBCP(Database Connection
Pool)的组件，另外，基于 Java 的开源连接池软件也很多，[这个链接](http://java-source.net/open-source/connection-pools)你可以看到这些描述。

## 复制扩展

复制可以使得数据库数据存在多个节点上，提高查询性能，至于写，Postgres-XC 项目这致力于这点。

### 热备

PostgreSQL 从 8.2 版本开始就已经内置了备机(standby)特性，它允许创建主备(master/standby)节点,备机定期从主机接收数据库更新日志并应用这些日志。
如果主机因为各种原因失效，备机可以快速替代主机。

更深入理解热备，需要我们先了解下面的一些术语：

Write-ahead log(WAL)

: :
预写日志，每个 WAL 大小为 16MB。如果有两台数据库开始都是一样，然后应用相同的 WAL 后，他们还是相同的，因为 WAL 包含了所有改变的数据。

Base backup

: 包括所有数据库崩溃后能够完整恢复的数据库备份。它一般使用 pg_start_backup 和 pg_stop_backup 指令来进行悲愤，它等于把整个数据库以及在备份期间有所改变而写入 WAL 文件全部备份一套。

Point-in-time recovery(PITR)

: 如果你有基本备份(base
backup)以及一系列 WAL 文件，那你就应用基础备份以及一部分 WAL 文件从而实现按时点恢复特性。

File-based log shipping

: 如果你数据库做一个基础备份(base
backup)，然后把所有的 WAL 文件传输到另外一个服务器上，那么新的服务器可以和原来那台保持同步

standby

: 有完整的基础备份(base
backup)以及机遇文件的日志流传输，这个系统称之为备机(standby),只要 WAL 可以持续传输，standby 可以和主机保持同步。

Failover

: 把备机从恢复模式切换到 active 状态，我们把这个过程称为失效切换。

### 总结

**Program** **Relication Method** **Synchronization**

---

WAL Shipping Master/Slave Async
Slony Master/Slave Async
Londiste Master/Slave Async
Mammoth Master/Slave Async
Bucardo Master/Slave or Master/Master Async
Rubyrep Master/Slave or Master/Master Async
PgCluster Master/Master Sync
Postgres-XC Master/Master Sync
pgpool-II Statement-based middleware Sync

上述这么多主/从复制解决方案中的主要不同在于数据库的修改数据如何传递

- PostgreSQL 9.0 的 Hot Standby 主要用于高可用失效切换

- WAL 用于为备机数据库创建修改的日志块，它只能完整应用，不用应用 WAL 子集

- 使用日志传输复制会在主机和备机中产生归档延迟，虽然这点在 Postgresql
  9.0 里使用流复制(Stream Replication)特新得以优化，但还无法完全避免。

- Hot
  Standby 系统可以针对快速失效切换，长时间查询执行以及对主机最小影响三个方面进行优化。但同一时刻只能优化这三种的两种。

- 基于语句的数据库复制反而更灵活，还能用来升级版本。但有些语句并不容易复制，而且相比 WAL 传输有更高的开销。

- Slony 和 Londiste 都是使用触发器提取更新语句的成熟解决方案。

- pgpool-II 可以用来做多从节点的只读负载均衡，同时也可以做某些同步复制

- Bucardo 支持多主结构，不管这些节点是否一直都在线。

## 数据分区

当表一个自身比物理内存还大时，即便索引很好，查询该表的时间也会显著增加。此时一个自然的办法是把这个表划分成几个小的相关表。应用程序不修改修改，还是查询原来的表。
但到数据库层，则只是查询该表的子集而不是全部。

### 范围分区

我们使用第[10](#sec:query optimization){reference-type="ref"
reference="sec:query optimization"}章里提到的 Dell Store 2
数据库做例子，考虑 orders 表的结构：

```sql
dellstore2=# \d orders
 Table "public.orders"
   Column    |     Type      |                        Modifiers
-------------+---------------+----------------------------------------------------------
 orderid     | integer       | not null default nextval('orders_orderid
                                _seq'::regclass)
 orderdate   | date          | not null
 customerid  | integer       |
 netamount   | numeric(12,2) | not null
 tax         | numeric(12,2) | not null
 totalamount | numeric(12,2) | not null
Indexes:
    "orders_pkey" PRIMARY KEY, btree (orderid)
    "ix_order_custid" btree (customerid)
Foreign-key constraints:
    "fk_customerid" FOREIGN KEY (customerid) REFERENCES customers(customerid)
    ON DELETE SET NULL
Referenced by:
    TABLE "orderlines" CONSTRAINT "fk_orderid" FOREIGN KEY (orderid)
    REFERENCES orders(orderid) ON DELETE CASCADE
```

假设这个表随着时间推移，该表迅速膨胀，比如达到一亿条记录，或者说其大小超过了物理内存，此时我们要考虑对此表进行分区。
这里重点要考虑的是依据那个字段进行分区，比如这里有两个字段可以分区，一个是 orderid，一个是 orderdate。似乎两个都可行。
但是考虑到一个问题，订单只保留一段时间，比如 1，2 年。老的订单就需要删除。当大量的老订单删除时，必然就需要执行 vacuum。从而导致开销增加。
如果是依据 orderdate 分区，当需要删除老数据时，直接 DROP 掉包含老数据的分区就好了，这样效率就高很多。
因此这里选择 orderdate 作为分区字段更好。

接下来就要计算一个分区里应该包含多少数据，首先看看这个表的相关信息：

```sql
dellstore2=# select min(orderdate),max(orderdate) from orders;
    min     |    max
------------+------------
 2004-01-01 | 2004-12-31
(1 row)
```

作为演示数据，这里的数据只有 1 年的，因此我们按照月来分区。

```sql
create table orders_2004_01 (
   check (orderdate>= DATE '2004-01-01' AND orderdate < DATE '2004-02-01')
) inherits(orders);

...

create table orders_2004_12(
   check(orderdate>= DATE '2004-12-01' AND orderdate < DATE '2005-01-01')
) inherits(orders);
```

这些分区表只是继承了主表的字段，还需要增加索引，约束以及调整和主表相匹配的权限。

```sql
ALTER TABLE ONLY orders_2004_01
    ADD CONSTRAINT orders_2004_01_pkey PRIMARY KEY (orderid);
...

ALTER TABLE ONLY orders_2004_12
    ADD CONSTRAINT orders_2004_12_pkey PRIMARY KEY (orderid);

CREATE INDEX idx_orders_2004_01_custid ON orders_2004_01
    USING btree (customerid);

....

CREATE INDEX idx_orders_2004_12_custid ON orders_2004_12
    USING btree (customerid);

ALTER TABLE ONLY orders_2004_01 ADD CONSTRAINT fk_2004_01_customerid FOREIGN KEY (customerid)
    REFERENCES customers(customerid) ON DELETE SET NULL;

...

ALTER TABLE ONLY orders_2004_12 ADD CONSTRAINT fk_2004_12_customerid FOREIGN KEY (customerid)
   REFERENCES customers(customerid) ON DELETE SET NULL;
```

到此为止，表结构都已经有了，下一步就是确保插入到主表的数据都插入到对应的分区表中，我们建议使用触发器来实现这个功能：

```sql
CREATE OR REPLACE FUNCTION orders_insert_trigger()
RETURNS TRIGGER AS  $body$
BEGIN
IF    ( NEW.orderdate >= DATE '2004-12-01' AND
          NEW.orderdate < DATE '2005-01-01' ) THEN
         INSERT INTO orders_2004_12 VALUES (NEW.*);
 ELSIF ( NEW.orderdate >= DATE '2004-11-01' AND
          NEW.orderdate < DATE '2004-12-01' ) THEN
         INSERT INTO orders_2004_11 VALUES (NEW.*);

...

 ELSIF ( NEW.orderdate >= DATE '2004-00-01' AND
         NEW.orderdate < DATE '2004-01-01' ) THEN
         INSERT INTO orders_2004_11 VALUES (NEW.*);

ELSE
         RAISE EXCEPTION 'Error in orders_insert_trigger():    date out of range';
     END IF;
     RETURN NULL;
 END;
 $body$
 LANGUAGE plpgsql;
```

这个触发器是静态的，每次都会执行相同的语句，而且不容易扩展，我们可以改造为动态触发器，类似下面这样：

```sql
CREATE OR REPLACE FUNCTION orders_insert_trigger()
RETURNS TRIGGER AS $body$
DECLARE
    ins_sql TEXT;
BEGIN
ins_sql :=
        'INSERT INTO orders_'|| to_char(NEW.orderdate, 'YYYY_MM') ||
        '(orderid,orderdate,customerid,netamount,tax,totalamount)
          VALUES ' ||
        '('|| NEW.orderid || ',' || quote_literal(NEW.orderdate) || ','
           || NEW.customerid ||','||
        NEW.netamount ||','|| NEW.tax || ',' || NEW.totalamount || ')';
    EXECUTE ins_sql;
    RETURN NULL;
END
$body$
LANGUAGE plpgsql;
```

而后把这个触发器应用到主表 orders 上:

```sql
CREATE TRIGGER insert_orders_trigger
    BEFORE INSERT ON orders
    FOR EACH ROW EXECUTE PROCEDURE orders_insert_trigger();
```

除了用触发器这种方法来重定向数据的插入外，我们还可以利用规则(rule)来做到这点，就像下面这样：

```sql
CREATE RULE orders_2004_01_insert AS
  ON INSERT TO orders WHERE
         ( orderdate>=DATE '2004-01-01' AND orderdate < DATE '2004-02-01' )
  DO INSTEAD
         INSERT INTO orders_2004_01 VALUES(NEW.*);
```

在数据批量加载时，规则相比触发器更快。不过使用规则的是个弱点就是无法使用 COPY 指令来加载数据。

### 现场迁移到分区表

现在所有的分区表以及表上索引、约束以及主表上需要的触发器都已经创建好，但是目前所有的数据都还在主表上，如何把数据分布到分区表呢？

有两个办法，一个最容易也是最简单的办法是先把主表上的数据导出，而后清空主表，然后导入刚才导出的数据。这个做法需要一定的停服务时间。

另外一个办法是考虑使用更新触发器，通过更新主表上的所有数据，从而使得数据重新分配。这个方法的流程和使得插入数据重定向方法类似，首先创建一个主表更新函数。
而后在主表上针对更新创建触发器，类似下面这样：

```sql
CREATE OR REPLACE FUNCTION orders_update_trigger()
RETURNS TRIGGER AS $body$
BEGIN
   DELETE FROM orders WHERE OLD.orderid=orderid;
    INSERT INTO orders values(NEW.*);
    RETURN NULL;
END;
$body$
LANGUAGE plpgsql;

CREATE TRIGGER update_orders
     BEFORE UPDATE ON orders
     FOR EACH ROW
     EXECUTE PROCEDURE orders_update_trigger();
```

而后我们把对主表的更新语句放在一个事物里，就像下面这样：

```sql
dellstore2=# BEGIN;
BEGIN
dellstore2=# SELECT count(*) FROM orders;
 count
-------
 12000
(1 row)

dellstore2=# SELECT count(*) FROM  orders_2004_01;
 count
-------
     0
(1 row)

dellstore2=# SELECT count(*) FROM  orders_2004_12;
 count
-------
     0
(1 row)

dellstore2=# UPDATE orders SET orderid=orderid;
UPDATE 0
dellstore2=# SELECT count(*) FROM orders;
 count
-------
 12000
(1 row)

dellstore2=# SELECT count(*) FROM orders_2004_01;
 count
-------
  1000
(1 row)

dellstore2=# SELECT count(*) FROM orders_2004_12;
 count
-------
  1000
(1 row)

dellstore2=# COMMIT;
COMMIT
```

确保所有的数据都已经重新分布后，记得做善后工作

```sql
dellstore2=# CLUSTER orders;
ERROR:  there is no previously clustered index for table "orders"
STATEMENT:  CLUSTER orders;
ERROR:  there is no previously clustered index for table "orders"
dellstore2=# DROP TRIGGER update_orders ON orders;
DROP TRIGGER
dellstore2=# DROP FUNCTION orders_update_trigger();
DROP FUNCTION
```

### 分区查询

到此为止，数据都在分区表里了，此时应该更新攻击。同时也应该确认 constraint_exclusion 特性是否打开:

```sql
dellstore2=# ANALYZE ;
ANALYZE
dellstore2=# SHOW constraint_exclusion ;
 constraint_exclusion
----------------------
 partition
(1 row)
```

排他性约束可以让查询计划避免包含进并不满足查询要求行的分区表进来。PostgreSQL
8.4 及以后版本默认是打开的，之前则是关闭。

通过对主表的查询测试，我们可以看到查询计划已经在分区表上起作用了:

```sql
dellstore2=# EXPLAIN ANALYZE SELECT * FROM orders;
  QUERY PLAN
------------------
 Result  (cost=0.00..228.00 rows=12001 width=30) (actual time=0.010..4.648
                                                      rows=12000 loops=1)
   ->  Append  (cost=0.00..228.00 rows=12001 width=30)
                            (actual time=0.009..2.464 rows=12000 loops=1)
         ->  Seq Scan on orders  (cost=0.00..0.00 rows=1 width=30)
                            (actual time=0.002..0.002 rows=0 loops=1)
         ->  Seq Scan on orders_2004_01 orders  (cost=0.00..19.00 rows=1000 width=30)
            (actual time=0.007..0.118 rows=1000 loops=1)
         ->  Seq Scan on orders_2004_02 orders  (cost=0.00..19.00 rows=1000 width=30)
            (actual time=0.006..0.111 rows=1000 loops=1)
        ...
         ->  Seq Scan on orders_2004_11 orders  (cost=0.00..19.00 rows=1000 width=30)
             (actual time=0.003..0.096 rows=1000 loops=1)
         ->  Seq Scan on orders_2004_12 orders  (cost=0.00..19.00 rows=1000 width=30)
             (actual time=0.003..0.105 rows=1000 loops=1)
 Total runtime: 5.339 ms
(16 rows)

dellstore2=# EXPLAIN ANALYZE SELECT * FROM orders WHERE orderdate='2004-11-16';
  QUERY PLAN
---------------
 Result  (cost=0.00..21.50 rows=36 width=30) (actual time=0.012..0.136 rows=35 loops=1)
   ->  Append  (cost=0.00..21.50 rows=36 width=30)
                (actual time=0.011..0.127 rows=35 loops=1)
         ->  Seq Scan on orders  (cost=0.00..0.00 rows=1 width=30)
                (actual time=0.001..0.001 rows=0 loops=1)
               Filter: (orderdate = '2004-11-16'::date)
         ->  Seq Scan on orders_2004_11 orders  (cost=0.00..21.50 rows=35 width=30)
                (actual time=0.009..0.122 rows=35 loops=1)
               Filter: (orderdate = '2004-11-16'::date)
               Rows Removed by Filter: 965
 Total runtime: 0.165 ms
(8 rows)
```

我们再看看针对 orderid 字段查询的例子：

```sql
dellstore2=# EXPLAIN ANALYZE SELECT * FROM orders WHERE orderid<2000;
  QUERY PLAN
-----------------------
 Result  (cost=0.00..258.00 rows=2011 width=30) (actual time=11.433..39.440 rows=1999 loops=1)
   ->  Append  (cost=0.00..258.00 rows=2011 width=30) (actual time=11.431..38.878 rows=1999 loops=1)
         ->  Seq Scan on orders  (cost=0.00..0.00 rows=1 width=30) (actual time=0.001..0.001 rows=0 loops=1)
               Filter: (orderid < 2000)
         ->  Seq Scan on orders_2004_01 orders  (cost=0.00..21.50 rows=1000 width=30)
              (actual time=11.429..14.192 rows=1000 loops=1)
               Filter: (orderid < 2000)
       ...
         ->  Seq Scan on orders_2004_12 orders  (cost=0.00..21.50 rows=1 width=30)
              (actual time=1.739..1.739 rows=0 loops=1)
               Filter: (orderid < 2000)
               Rows Removed by Filter: 1000
 Total runtime: 39.880 ms
(40 rows)
```

从结果来看，针对 orderid 的查询并没有太多优化，所有的子表都在扫描范围内。

### 常见的分区误区

- 没有打开 constraint_exclustion，从而导致每次查询都包含所有的分区表

- 在每个分区表上增加主表上已经有的索引或者约束时会报错

- 忘记给每个分区表赋予和主表相同的权限

- 查询语句没有包括分区字段的过滤，WHERE 子句需要用常量来过滤，当参数的查询根本就不会优化。
  函数里通过使用类似 CURREN_DATE 这样的语句修改过滤值也不会获得好的优化效果。一般来说，WHERE 子句尽可能简单。

- 针对分区的查询开销和分区数量成正比，如果只有 10 ～ 20 个分区表，那开销不明显，如果上百个，那就很明显了，因此保持你的分区表数量在 2 位数之内时最好的性能。

- 如果你手工对主表执行 vacuum/analyze 操作，它并不会执行到分区表上，因此你需要针对每个分区表做同样的操作。

- 插入超过日期范围(比如 2005 年的)的数据会导致函数报错，类似 ERROR:
  relation \"orders_2013_03\" does not exist。

### 用 PL/Proxy 做水平分区

[PL/Proxy](http://pgfoundry.org/projects/plproxy/)的理论是认为把一个大表分成多个子表会比将大表分到相同的多个服务器上更有效率。

PL/Proxy 是 Skype 设计用来满足数据库扩展所需，它其中的一个目标是一次可以服务 10 个用户请求。它是用存储过程语言实现的数据库分区系统。

PL/Proxy 的一个基本假设是通过访问函数（也就是存储过程）来屏蔽直接访问数据库。比如，假定你需要抓取 username 字段来唯一标识一个用户，你可以调用
一个返回 username 字段的函数而不是查询表。这个设计方式流行的另外一个原因是，它使得当需要重构数据库时，对应用程序的影响最小。只要那些称之为"API"的函数是稳定的，
那么函数映射到重构后的数据库就要简单得多。

接下来要做的是判断用哪种方法把负载分片，从技术上来说，就是判断用哪个字段作为分区依据。

PL/Proxy 的配置安装有点复杂，感兴趣的可以看这个这个链接：<http://plproxy.projects.postgresql.org/doc/tutorial.html>

### 散列生成

上面提到 PL/Proxy 的一个标准工作就是表分区，因为是水平分区，我们需要考虑记录和子表如何对应。一般我们采用对特定字段做 hash 算法的方式映射到子表中。
我们可以用 PostgreSQL 自带的 hashtext 函数来实现 hash 算法。它可以接受任何文本作为输入，而后产生一个整数作为 hash
code。如果你把 hash code 和一个数做位与操作，则可以产生最低的几位数字。
为了实现这点，分区表的数量必须是 2 的幂数(2,4,8,16 等)，然后把 hash
code 和比分区数少 1 的数进行位与。比如你有 2 个分区，则为\"&
1\",4 个分区则是\"& 3\"，以此类推。

我们举一个例子，把前 10 个自然数映射到 4 个分区表中：

```sql
select s,hashtext(s::text) & 3 as hash from generate_series(1,10) as s;
 s  | hash
----+------
  1 |    1
  2 |    2
  3 |    0
  4 |    0
  5 |    2
  6 |    0
  7 |    1
  8 |    0
  9 |    0
 10 |    3
(10 rows)
```

从 hash 例可以看出，这个 10 个自然数被映射到(0,1,2,3)集合里。

hash 要考虑的另外一个情况是，对集合到子集的映射尽可能平均，因为我们可以看看前 10000 个自然数映射到 4 个分区里的统计情况：

```sql
select hashtext(s::text) & 3 as hash,count(s) as count from generate_series(1,2000)
 hash | count
------+-------
    0 |  2550
    1 |  2411
    2 |  2531
    3 |  2508
(4 rows)
```

从这个结果来看，子集元素的数量还是很均等的。

要注意的是，PostgreSQL
8.3 版本的 hashtext 函数和 8.4 版本的 hashtext 输出结果不同，因此当你从 8.3 版本升级时，则需要考虑这点。

### 分片

分片(Sharding)是一种水平分区的形式，可以用 PL/Proxy 来实现。它在每个节点上都有分片数据结构，因此这些节点中的每个都可以独立操作。
这一般是 shared-nothing 架构所涉及到的方案，每个分片节点都可以独自响应一个查询请求--只要所查询的数据在其中。

分片设计的复杂处包括如何协同以及分片节点如何更新，它一般和特定的应用程序相关。比如 Skype 就使用了 Londiste 复制程序。

### 总结

PostgreSQL 里的每一种分区方案都都一定的手工配置工作要求。因此，做这些操作前，你需要获得足够的信息来判读使用哪种分区方案以及如何分区。

这里有些通用指导建议：

- 只有数据库表或者表活跃部分超过服务器物理内存时，分区才有意义。

- 选择哪个字段作为分区依据需要仔细评估针对该表的所有查询，以确保他们针对该字段都有选择性。

- 每个分区表都需要手工创建，包括索引、约束以及(或)表权限。

- 重定向 INSERT，UPDATE，DELETE 语句到分区表的最容易办法是使用触发器。

- 从未分区表的数据现场迁移到分区表中是可能的，虽然会带来临时的磁盘使用空间翻倍的现象。

- 对于分区表，需要打开 constraint_exclustion 特性，查询才会正常工作

- WHERE 子句需要简单值才能使 constraint exclustion 工作

- 通过分区可以改进某些数据库维护操作。对单个分区表执行 REINDEX 会更快，相比通过 DELETE 删除老的数据，通过 DROP 老的分区表来删除数据几乎时瞬时的，

- 使用前自动创建分区表需要慎重思考，需要考虑到条件竞争情况

- 可以有效使用的分区表数上限是两位数。超过 100 个分区表的情况下，查询会需要很明显的开销来评估这些分区约束条件

- PL/Proxy 可以用来有效的把数据拆分到多个节点，且扩展节点无限制，它也可以用来部署 shared-nothing 架构

## 避免常见问题

PostgreSQL 官方 wiki 上有一个[FAQ](http://wiki.postgresql.org/wiki/Category:FAQ)列表，列出了一些常见的问题，有些我们会在下面提到。

### 灌数

数据灌入有两种不同的情形。第一种在一个空的数据库(表)里灌入。另外一种是数据库(表)里已经有数据了，然后再灌入。两种有一些细微的区别。
比如用于第一种灌数情形的删除索引和约束就不适合第二种情形。我们分别介绍。

#### 加载方法

数据灌入的首选方法是用 PostgreSQL 特有的 COPY 指令，这是插入大量行记录的最快方法。
当然你也可以使用传统的 INSERT 指令，但即便用上 BEGIN/COMMIT 方式包裹多条 INSERT 语句，速度也不如 COPY 来的快。

下面是从慢到快的一个数据插入方法：

1.  一次插入单个记录

2.  一次插入多条记录

3.  一次使用单个 COPY 指令，这也是标准 pg_restore 所用方法

4.  多条插入指令并行执行，每条插入指令包括多条记录

5.  一次使用多个 COPY 指令。这是并行 pg_restore 所用方法

#### 外部灌数程序

使用 COPY 指令需要考虑的一点是，只要出现错误，它就会终端所有的工作。想象你一下就因为最后一条记录有问题，从而导致你前面大量的数据插入都失效，这是不能接受的。

假定你是从外部数据源倒入数据，那就应该考虑导入操作应该可以持续工作，然后把因为错误拒绝插入的数据另外保存起来。[pgloader](http://pgfoundry.org/projects/pgloader/)就
可以做到这点，虽然它不如 COPY 来得快，但对于外部数据格式并不那么严格时，这是最好的方式。

另外一个用于某些特定场景下的灌数程序是[pg_bulkload](http://pgbulkload. projects.postgresql.org/)，它有时甚至比 COPY 还要快，这得意于它采取积极的高性能特性，比如绕过 WAL。
它也可以处理坏的数据行。

#### 为灌数调优

为灌输而关闭表上的所有索引以及约束只最需要做的事情，灌输后在创建索引会更快，更好的磁盘碎片。灌输后做约束检查要快很多。

postgresql.conf 里的几个参数也可以在灌数时做临时调整，可以加快灌数的速度：

- maintenance_work_mem:
  尽可能加大该值，对于超过 16G 内存的服务器，配置 1GB 的该参数就不合道理。CREATE
  INDEX ，ALTER TABLE ADD FOREIGN KEY 都需要它。

- checkpoint_segments:
  要比正常使用时的值大很多，因为崩溃后恢复时间以及磁盘空间在这个时间段并不是需要关注的事情。通常设置为 128-256。同样可以加大 checkpoint_timeout
  的值，设置为 30m 是合理的

- autovacuum: 灌数时关闭该特性

- wal_buffers: 它很容易成为灌数的瓶颈，设置到其最大值 16MB。

- synchronous_commit:
  关闭该参数，关闭后如果因为崩溃而导致丢失某些事务，简单清空崩溃前最后灌入的表，重新灌入就好了

- fsync:
  该参数只有在灌数时可以考虑关闭。带来的主要问题是当数据库崩溃时，数据可能损坏，但不是致命错误。只需要完全重新灌数就好

- vacuum_freeze_min_age: 设置为比 0 小的参数

#### 跳过 WAL 来加速

假定一个灌数代码类似如下：

```sql
BEGIN;
CREATE TABLE t ...;
COPY t FROM ...;
COMMIT;
```

缺省情况下，上述代码执行不会产生任何 WAL 记录，因此执行起来更快。同理，如果你用 TRUNCATE 而不是 DELETE 清空一个表也能获得提速。

但是，如果你设置了 archive_command 参数来获得 PITR 特性，那么上述优化就生效了。

#### 重建索引和增加约束

索引重建很可能成为 CPU、内存以及磁盘密集型操作。如果你有个大磁盘阵列，那么并行运行两个或以上重建指令是非常合理的。
如前所述，增加 shared_buffers,checkpoint_segment,checkpoint_timeout,maintenance_work_mem 参数值可以提高性能。

#### 并行恢复

PostgreSQL
8.4 引入了并行恢复机制，它允许你分配服务器上的多个 CPU 核来运行专门的数据加载进程。使用并行版本的 pg_restore，很简单你只需要指定你一次想运行
多少个"jobs"就好了，它对数据加载以及索引重建和增加约束都有提速效果。

#### 灌数后清理工作

灌数完毕，索引重建完成，约束增加完毕。把服务器上线之前还有两个维护工作要做。
第一个必须要做是对每个数据库执行 ANALYZE 指令。确保统计信息有效可用。同时也可以考虑运行数据库级的 VACUUM。当然你直接运行 VACUUM
ANALYZE 会更好。
第二个要做的事情就是把 postgresql.conf 里修改过的参数恢复到正常设置。

### 常见性能问题

很多性能问题都是来自糟糕的设计或者实现，使得 PostgreSQL 无法从根本上很好的运行。下面我们列举一些 PostgreSQL 新手可能遇到的常见性能问题：

#### 统计行数

应用程序里大部分都有类似下面的 SQL 语句来计算一个表里有多少行：

```sql
    SELECT count(*) FROM t;
```

有些数据库可能执行这个很快，通常他们把这个信息保存到一个索引里或者类似的结构里。
不幸的是，PostgreSQL 没这么干，它需要执行全表扫描才能获得结果，这就相当慢了。
对此，PostgreSQL 提供了两个替代方法。第一个方法是对于那些并不需要精确的表行数而言，我们可以从系统表里查询相关信息，比如：

`SELECT reltuples FROM pg_class where relname='t'`

如果运行该 SQL 语句之前刚好运行过 VACUUM 或者 ANALYZE，则这个结果是准确的，而不是估计值。

如果你有同名的表，那么就加上名字空间限定，类似下面这样：

```sql
SELECT reltuples
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
WHERE C.relname='t' and N.nspname = 'public';
```

第二办法是在表上创建一个触发器，跟踪表上数据的删除和增加操作，从而把统计行数记录下来。
具体的可以参考这个例子：<http://wiki.postgresql.org/wiki/Slow_Counting>

#### 外键开销高

当更新语句里涉及到外键，其时间开销很大，特别是每次只做一行更新时。对于批量更新，可能的一个性能提升办法时把外键约束标记为 DEFERRABLE，缺省是
NOT
DEFERRABLE。不过这种设置只对把外键和相似约束作为触发器实现时才有效，而对于 CHECK 以及 UNIQUE 限定则无能为力。后者要求立刻处理，不能延缓。

即便对于约束延缓，也有两种情况。如果检查标记为 INITIALLY
IMMEDIATE，那也会立刻处理，而不是在事务的最后来处理。此时你可以修改约束为 INITIALLY
DEFERRED，这样可以 确保检查在事务最后处理。

当检查标记为 DEFERRABLE，它同时为 INITIALLY IMMEDIATE 时，你可以使用 SET
CONSTRAINTS 指令修改部分会话。其标准做法是，把多条插入或更新语句放在
一个事务里，且事务开头标记所有约束为延缓，类似如下：

    BEGIN;
    SET CONSTRAINTS ALL DEFERRED;
    [ update or insert statements ]
    COMMIT;

这种处理方式对批量更新/插入可以提升效率，但是如果量很大，则在做 COMMIT 时，有可能会阻塞服务一段时间，从而导致无法对外响应。

#### 触发器内存使用

如果一个表上有 AFTER 类型触发器，那么每个插入语句都会在一个内存列表里，如果插入的数量很大，比如批量加载时，就有可能出现 OOM，或者达到系统设置的内存极限。
因此在批量倒入场景下，要注意 AFTER 类型触发器。

### 数据库性能分析

有时要找出你的代码为什么不能有效的工作的办法时深入到数据库本身，并查到有瓶颈的代码。我们通过一些工具来做到这点：

#### gprof

gprof 是一个标准的 GNU 性能分析工具，可以在大部分类 UNIX 系统上运行，编译 PostgreSQL 时，增加\--enable-profiling 选项，它会产生一个 gmon.out 文件，这根据可以用于 gprof 来做性能分析。

gprof 的主要问题时它在跟踪到可加载库的函数时会出现已知问题。另外，它也很能根据分析信息来获得下层操作系统正在做什么。因此，综合这两个限制，gprof 只适合做一些简单和单纯的数据库操作性能分析。

#### OProfile

OProfile 可以分析任何在 Linux 服务器上运行的程序，是一个非常棒的工具。也是很多在 Linux 写的码农们的首选工具。<http://wiki.postgresql.org/wiki/Profiling_with_OProfile>介绍的很详细。

#### Visual Studio

Windows 平台的最佳选择(或唯一选择？)，从 PostgreSQL 8.4 以后有效。

#### DTrace

Sun 公司在 2004 年发布的 Solaris
10 中引入 DTrace，可能时最全面的性能分析工具。

PostgreSQL 编译时需要使用\--enable-dtrace 选项才能使用 DTrace，对于如何使用 DTrace，恐怖可以用一本书的容量来介绍。我们可以参考下面的两个链接：

- <http://www.postgresql.org/docs/current/interactive/dynamic-trace.html>

- <http://wiki.postgresql.org/wiki/DTrace>

#### Linux SystemTap

SystemTap 是 Linux 平台性能分析工具，其功能类似 DTrace。对使用了\--enable-dtrace 编译的 PostgreSQL，SystemTap 可以使用。

目前 Fedora,RHEL 系统发型班办法都能支持 SystemTap,但 Debian/Ununtu 则可能会有些问题。

### 与版本有关的性能特性

书中描述了各种和版本有关的性能特性，这里时摘录于 9.0 有关的特性。

#### 复制

- 异步主从复制已经内置，且不需要配置太多额外脚本

- 复制系统还可以配置为 HotStandby 用于只读查询

- 控制 WAL 归档参数分开，除了已经存在的 archive_mode 和 archive_command 以外，新增了 wal_level 参数用于设置允许写多少信息到 WAL 里

- 复制过程的监控可以通过 standby 服务器使用 pg_last_xlog_receive_location()和 pg_last_xlog_replay_location()函数实现

- contrib/pg_archivecleanup 工具可以用于清除老的 WAL 归档文件。该程序可以被 archive_cleanup_command 参数调用

#### 查询和分析

- EXPLAIN 可以输出为 XML，JSON 或 YAML 格式，方便外部工具分析

- EXPLAIN （BUFFERS） 可以监控某个特定查询的实际 I/O,而不是显示成本

- 窗口函数新增了诸如 PRECEDING，FOLLOWING 选项，以及用 CURRENT
  ROWS 开始的选项

- 外联接所引用到的表如果并没用于满足查询要求，则会从查询计划中移去以提升性能
  。一般这类查询语句由 ORM 软件生成。

- 每个表空间可以通过 ALTER
  TABLESPACE 来分配自己的顺序和随机页查询代价，对于不同表空间分配在不同速度的存储介质上这种情况，可以获得更好的调整

- 可以通过使用 ALTER
  TABLE 来手工调整一个字段不同值的统计信息，当自动评估有问题时，这种方式可以提升效率

- 使用 IS NOT NULL 的查询语句可以使用索引

- 基因查询优化器(Genetic Query Optimizer a.k.a
  GEQO)现在可以强制使用固定的随机种子值

- GIN 索引使用自平衡红黑树(self-balancing Red-Black trees)

#### 数据库开发

- PL/pgSQL 语句现在缺省安装到所有数据库，可以删除

- 触发器可以附着在特定的字段上，以避免不必要的调用

- LISTEN/NOTIFY 机制使得数据库客户端直接传递消息更快，而且可以发送"payload"字符串信息，这使得使用数据库自身创建跨客户端消息系统成为可能

- 应用程序能够使用 application_name 来设置，使得 pg_stat_activity 信息里显示出应用程序名字

- 唯一约束违例发生时给出的错误信息里包括的设计到的值，这使得定位到底时那里导致发生错误变得容易

- 约束检查可以标记为 DEFERRABLE

- 新引进排他性约束(exclusion constraints)，和唯一性约束类似

- contrib/hstore 模块提供了一个低开销的键值对(key/value)存储类型

#### 配置和监控

- 服务器参数可以被用户/数据库组合调整

- 当 postgresql.conf 文件被被 pg_ctl
  reload 或发送 SIGHUP 信号给服务而重载时，服务器日志会注意到哪些参数被修改了

- 使用 pg_stat_reset_shared('bgwriter')函数可以重置后台写统计

- 使用 pg_stat_reset_single_table_counters()和 pg_stat_reset_single_function_counters()可以分别单独针对表和函数计数器进行重置

- log_temp_files 参数可以用 KB 单位来指定

- log_line_prefix 可以追踪诸如错误消息里设置的 SQLSTATE 值，这就使得日志可以更好的分析导致错误发生的确切代码

#### 工具

- pgbench 可以运行多进程/多线程模式性能测试

- contrib/auto_explain 模块可以输出查询语句以及它的执行计划

- contrib/pg_stat_statements 可以手机查询日志数据包括每个语句所有的活动缓存信息

#### 内部

- contrib/pg_upgrade 工具可以从早期的 8.3/8.4 版本平滑升级到 9.0,不需要导出导入数据，这中方式一般称为就地更新(\"in-place
  upgrade\")

- VACUUM
  FULL 使用了早期版本中 CLUSTER 中使用的重建机制。它需要更多的磁盘空间，但更快而且可以避免索引膨胀

- 编译时默认使用--enable-thread-safety。该选项允许多线程客户端程序使用数据库客户端库来编译，从而安全运行。早起版本默认不开启该特性

- 数据库有自己更好的办法处理 OOM 的问题

- 有几个指令现在可以通过使用()包括的列表来传递选项，而不是传统的 SQL 语法。EXPLAIN，COPY，VACUUM 目前都可以使用这个特性

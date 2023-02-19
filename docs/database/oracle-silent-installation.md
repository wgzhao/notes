# Oracle静默安装

Oracle 数据库默认采取图形界面安装，但是对于没有图形界面或者很难启用图形界面的情况下，可以采取指定应答文件的方式来安装。

应答文件有两种获取方式，一种是在某一台安装过的机器上，当时保存过应答文件，然后拿过来修改就可以了。
另外一种就是用安装介质中提供的应答文件模板进行修改。模板在安装介质的_respone_目录下。 该目录下有三个文件，从名字可以看出对应的功能，这里我们针对 `db_install.rsp` 文件进行修改，类似如下：

```ini
--8<-- "db_install.rsp"
```

保存该文件，比如 _/tmp/db-install.rsp_ 

接下里需要做以下几个步骤：

0. 安装 Oracle JDK `rpm -ivh jdk1.8.0u91*.rpm`
1. 创建对应的账号和用户组: `groupadd oinstall && useradd -g oinstall oracle`
2. 安装必要的软件包 `yum install gcc gcc-c++ gcc-c++-devel glibc-devel libgcc libgcc-devel libaio libaio-devel compat-libstdc++-33 libstdc++ libstdc++-devel elfutils-libelf-devel ksh libaio`
3. 开始进行静默安装 `./runInstaller -silent -responseFile /tmp/db-install.rsp -jreLoc /usr/java/jdk1.8.0_91/jre -ignoreSysPrereqs -ignorePrereq`

安装完毕后，配置 oracle 账号的顺境变量。以及增加 Oracle 账号的打开文件数等

```bash
cat ->/etc/security/limits.d/oracle.conf
oracle   soft   nofile    8192
oracle   hard   nofile    65536
oracle   soft   nproc    16384
oracle   hard   nproc    16384
oracle   soft   stack    10240
oracle   hard   stack    32768
oracle   hard   memlock    134217728
oracle   soft   memlock    134217728
^D
```


# rpm 包管理基本技能

[TOC]

## 讨论点

- RPM 包管理基本操作
- 编译自己的 RPM 包
- 创建可下载的 RPM 包
- 高级 RPM 打包技术



## RPM 包管理基本操作

- RPM means Redhat Package Management
- RedHat 系列， Fedora 系列，CentOS 系列以及其他部分国产Linux操作系统 使用 RPM
- 由软件包以及一套脚本组成

### 常用 RPM 命令

- 安装
  - `rpm -ivh <package name>`
- 更新
  - `rpm -Uvh <package name>`
- 刷新
  - `rpm -Fvh <package name>`
- 删除
  - `rpm -e <package name>

### RPM 包查询

- `rpm -q[option] <package name>`

- `-qi`: 查询包基本信息

  ```shell
  $ rpm -qi kernel
  Name        : kernel
  Version     : 3.10.0
  Release     : 957.el7
  Architecture: x86_64
  Install Date: Wed 10 Jun 2020 06:01:23 PM CST
  Group       : System Environment/Kernel
  Size        : 66205689
  License     : GPLv2
  Signature   : RSA/SHA256, Fri 05 Oct 2018 08:48:54 AM CST, Key ID 199e2f91fd431d51
  Source RPM  : kernel-3.10.0-957.el7.src.rpm
  Build Date  : Fri 05 Oct 2018 05:13:25 AM CST
  Build Host  : x86-040.build.eng.bos.redhat.com
  Relocations : (not relocatable)
  Packager    : Red Hat, Inc. <http://bugzilla.redhat.com/bugzilla>
  Vendor      : Red Hat, Inc.
  URL         : http://www.kernel.org/
  Summary     : The Linux kernel
  Description :
  The kernel package contains the Linux kernel (vmlinuz), the core of any
  Linux operating system.  The kernel handles the basic functions
  of the operating system: memory allocation, process allocation, device
  input and output, etc.
  ```

- `-ql`: 列出包包含的文件

  ```shell
  $ rpm -ql bind-utils
  /etc/trusted-key.key
  /usr/bin/dig
  /usr/bin/host
  /usr/bin/nslookup
  /usr/bin/nsupdate
  /usr/share/man/man1/dig.1.gz
  /usr/share/man/man1/host.1.gz
  /usr/share/man/man1/nslookup.1.gz
  /usr/share/man/man1/nsupdate.1.gz
  ```

- `-qf`: 查询文件属于哪个包

  ```shell
  $ rpm -qf /usr/bin/sudo
  sudo-1.9.5-3.el7.x86_64
  ```

- `--queryformat` 指定查询的输出格式

  ```shell
  rpm -q --queryformat "[%-50{FILENAMES} %10{FILESIZES}\n]" bind-utils
  /etc/trusted-key.key                                      750
  /usr/bin/dig                                           129472
  /usr/bin/host                                          117336
  /usr/bin/nslookup                                      117240
  /usr/bin/nsupdate                                       62304
  /usr/share/man/man1/dig.1.gz                             7339
  /usr/share/man/man1/host.1.gz                            3058
  /usr/share/man/man1/nslookup.1.gz                        2465
  /usr/share/man/man1/nsupdate.1.gz                        5229
  ```

  

- `--querytags` 列出有效的标签，用于 `queryformat`

  ```shell
  $ rpm -q --querytags bind-utils |head -n10
  ARCH
  ARCHIVESIZE
  BASENAMES
  BUGURL
  BUILDARCHS
  BUILDHOST
  BUILDTIME
  C
  CHANGELOGNAME
  CHANGELOGTEXT
  ```
  
- `-p` ：查询没有安装的包，直接查询 RPM 包本身

  ```shell
  $ rpm -qpi nginx-1.23.1-1.el7.ngx.src.rpm
  警告：nginx-1.23.1-1.el7.ngx.src.rpm: 头V4 RSA/SHA256 Signature, 密钥 ID 7bd9bf62: NOKEY
  Name        : nginx
  Epoch       : 1
  Version     : 1.23.1
  Release     : 1.el7.ngx
  Architecture: aarch64
  Install Date: (not installed)
  Group       : System Environment/Daemons
  Size        : 1132326
  License     : 2-clause BSD-like license
  Signature   : RSA/SHA256, 2022年07月19日 星期二 23时46分23秒, Key ID abf5bd827bd9bf62
  Source RPM  : (none)
  Build Date  : 2022年07月19日 星期二 23时21分32秒
  Build Host  : ip-10-1-17-54.eu-central-1.compute.internal
  Relocations : (not relocatable)
  Vendor      : NGINX Packaging <nginx-packaging@f5.com>
  URL         : https://nginx.org/
  Summary     : High performance web server
  Description :
  nginx [engine x] is an HTTP and reverse proxy server, as well as
  a mail proxy server.
  ```

- `-a` : 列出所有已经安装的包

  ```shell
  $ rpm -qa |head -n10
  iftop-1.0-0.21.pre4.el7.x86_64
  libref_array-0.1.5-32.el7.x86_64
  libpipeline-1.2.3-3.el7.x86_64
  python-magic-5.11-35.el7.noarch
  libsane-hpaio-3.15.9-3.el7.x86_64
  liberation-fonts-common-1.07.2-16.el7.noarch
  libblockdev-nvdimm-2.18-3.el7.x86_64
  sysvinit-tools-2.88-14.dsf.el7.x86_64
  python-blivet-0.61.15.72-1.el7.noarch
  python-chardet-2.2.1-1.el7_1.noarch
  
  $ rpm -qa --queryformat "[%-50{NAME}\n]" |head -n10
  iftop
  libref_array
  libpipeline
  python-magic
  libsane-hpaio
  liberation-fonts-common
  libblockdev-nvdimm
  sysvinit-tools
  python-blivet
  python-chardet
  ```
  
- `--scripts`:  查询包所内置的脚本

  ```shell
  $  rpm -q --scripts nginx
  postinstall scriptlet (using /bin/sh):
  
  if [ $1 -eq 1 ] ; then
          # Initial installation
          systemctl preset nginx.service >/dev/null 2>&1 || :
  fi
  preuninstall scriptlet (using /bin/sh):
  
  if [ $1 -eq 0 ] ; then
          # Package removal, not upgrade
          systemctl --no-reload disable nginx.service > /dev/null 2>&1 || :
          systemctl stop nginx.service > /dev/null 2>&1 || :
  fi
  postuninstall scriptlet (using /bin/sh):
  
  systemctl daemon-reload >/dev/null 2>&1 || :
  
  if [ $1 -ge 1 ]; then
      /usr/bin/nginx-upgrade >/dev/null 2>&1 || :
  fi
  ```
### RPM 包校验

- `rpm -V[option] <package name>`

- 通过指定的维度来比较目前软件包的文件状态和 RPM 数据库中文件状态的对比

- 可以比较文件大小，MD5 值，权限，类型，属组等

  ```shell
  $ rpm -qV  nginx
  S.5....T.  c /etc/nginx/nginx.conf
  ```

  | 标识 | 含义 |
  | ---- | ---- |
  | `5`  |      MD5校验和 |
  | `S`  |      文件大小 |
  | `L`  |      符号连接 |
  | `T`  |      修改时间 |
  | `D`  |      设备 |
  | `U`  |      用户 |
  | `G`  |      组 |
  | `M`  |      模式(包括许可和文件类型) |

### 签名校验

- `rpm {-K|--checksig} <package file>`

- 用来检验包的 `gpg/pgp` 签名是否正确

  ```shell
  # rpm -K net-tools-2.0-0.25.20131004git.el7.x86_64.rpm
  net-tools-2.0-0.25.20131004git.el7.x86_64.rpm: rsa sha1 (md5) pgp md5 确定
  
  # rpm -K zabbix-agent-5.0.25-1.el7.x86_64.rpm
  zabbix-agent-5.0.25-1.el7.x86_64.rpm: RSA sha1 (MD5) PGP md5 不正确
  ```

### 解压RPM包

- 通过把 RPM 包转成 cpio 流的方式间接解压

  ```shell
  # rpm2cpio net-tools-2.0-0.25.20131004git.el7.x86_64.rpm |cpio -id
  1849 块
  # tree -L 2 .
  .
  ├── bin
  │   └── netstat
  ├── sbin
  │   ├── arp
  │   ├── ether-wake
  │   ├── ifconfig
  │   ├── ipmaddr
  │   ├── iptunnel
  │   ├── mii-diag
  │   ├── mii-tool
  │   ├── nameif
  │   ├── plipconfig
  │   ├── route
  │   └── slattach
  └── usr
      ├── lib
      └── share
  
  5 directories, 12 files
  ```

## 编译自己的 RPM 包

简单讲述如何创建一个 RPM 包

### 配置编译环境

- 不要使用 `root` 帐号来做 rpm 包编译操作

- `$HOME/.rpmmacros` 配置（可选)

  - `%_topdir /path/to/rpm/build/env`
  - 默认是 `$HOME/rpmbuild`

- 创建需要的目录

  - `~/rpmbuild/BUILD`

  - `~/rpmbuild/RPMS/<arch>`

  - `~/rpmbuild/RPMS/noarch`

  - `~/rpmbuild/SOURCES`

  - `~/rpmbuild/SPECS`

  - `~/rpmbuild/SRPMS`

- `rpmdevtools` 提升效率

  - `rpmdev-setuptree`

| 目录    | 用途                                                         |
| ------- | ------------------------------------------------------------ |
| BUILD   | 编译时， 变量  `%buildroot` 指向该目录，他包含编译是的中间文件产出，包括编译日志等。 |
| RPMS    | 保存编译出来的二进制 RPM 包目录。它包含 CPU 架构有关的子目录，比如 `x86_64` 和 `noarch`. |
| SOURCES | 源代码和补丁文件所在目录  |
| SPECS   | spec 文件所在目录                        |
| SRPMS   | 当用 `rpmbuid` 来编译一个  SRPM 包而不是二进制 RPM 包时，会放在此目录。 |

### 创建 rpm 包

`rpmbuild` 是用来创建 rpm 包的命令，他提供了多种途径来编译 rpm 包，下面列出最常用的

| 参数 | 含义 |
| ---- | ---- |
| `-bp`  |  依据 `<specfile>` 从 `%prep` (解压缩源代码并应用补丁) 开始构建 |
| `-ba`  |  依据 `<specfile>` 构建源代码和二进制软件包 |
| `-bb`  |  依据 `<specfile>` 构建二进制软件包 |
| `-bs`  |  依据 `<specfile>` 构建源代码软件包 |
| `-ta`  |  依据 `<tarball>` 构建源代码和二进制软件包 |
| `-tb`  |  依据 `<tarball>` 构建二进制软件包 |
| `-ts`  |  依据 `<tarball>` 构建源代码软件包 |
| `--rebuild`  |  依据 `<source package>` 构建二进制软件包 |

- 需要软件源代码，补丁（如有）以及 `spec` 文件

### RPM Spec 文件

`spec` 文件内容分成以下内容

| 区域           | 定义                                                         |
| -------------- | ------------------------------------------------------------ |
| `%description` | A full description of the software packaged in the RPM. This description can span multiple lines and can be broken into paragraphs. |
| `%prep`        | Command or series of commands to prepare the software to be built, for example, unpacking the archive in `Source0`. This directive can contain a shell script. |
| `%build`       | Command or series of commands for actually building the software into machine code (for compiled languages) or byte code (for some interpreted languages). |
| `%install`     | Command or series of commands for copying the desired build artifacts from the `%builddir`(where the build happens) to the `%buildroot` directory (which contains the directory structure with the files to be packaged). This usually means copying files from `~/rpmbuild/BUILD` to `~/rpmbuild/BUILDROOT` and creating the necessary directories in `~/rpmbuild/BUILDROOT`. This is only run when creating a package, not when the end-user installs the package. See [Working with SPEC files](https://rpm-packaging-guide.github.io/#working-with-spec-files) for details. |
| `%check`       | Command or series of commands to test the software. This normally includes things such as unit tests. |
| `%files`       | The list of files that will be installed in the end user’s system. |
| `%changelog`   | A record of changes that have happened to the package between different `Version` or `Release`builds. |



每个区域部分都可以使用预先定义的指令，列举如下：

| SPEC 指定       | 定义                                                   |
| --------------- | ------------------------------------------------------------ |
| `Name`          | The base name of the package, which should match the SPEC file name. |
| `Version`       | The upstream version number of the software.                 |
| `Release`       | The number of times this version of the software was released. Normally, set the initial value to 1%{?dist}, and increment it with each new release of the package. Reset to 1 when a new `Version` of the software is built. |
| `Summary`       | A brief, one-line summary of the package.                    |
| `License`       | The license of the software being packaged. For packages distributed in community distributions such as [Fedora](https://getfedora.org/) this must be an open source license abiding by the specific distribution’s licensing guidelines. |
| `URL`           | The full URL for more information about the program. Most often this is the upstream project website for the software being packaged. |
| `Source0`       | Path or URL to the compressed archive of the upstream source code (unpatched, patches are handled elsewhere). This should point to an accessible and reliable storage of the archive, for example, the upstream page and not the packager’s local storage. If needed, more SourceX directives can be added, incrementing the number each time, for example: Source1, Source2, Source3, and so on. |
| `Patch0`        | The name of the first patch to apply to the source code if necessary. If needed, more PatchX directives can be added, incrementing the number each time, for example: Patch1, Patch2, Patch3, and so on. |
| `BuildArch`     | If the package is not architecture dependent, for example, if written entirely in an interpreted programming language, set this to `BuildArch: noarch`. If not set, the package automatically inherits the Architecture of the machine on which it is built, for example `x86_64`. |
| `BuildRequires` | A comma- or whitespace-separated list of packages required for building the program written in a compiled language. There can be multiple entries of `BuildRequires`, each on its own line in the SPEC file. |
| `Requires`      | A comma- or whitespace-separated list of packages required by the software to run once installed. There can be multiple entries of `Requires`, each on its own line in the SPEC file. |
| `ExcludeArch`   | If a piece of software can not operate on a specific processor architecture, you can exclude that architecture here. |

## 实战

从三种场景来编译 rpm 二进制包，

### 源代码准备

我们假定有一个自创的项目，源代码只包含一个 `cello.c`

内容如下：

```c
#include <stdio.h>

int main(void) {
    printf("Hello World\n");
    return 0;
}
```

构建 `Makefile` 文件



```mak
cello:
        gcc -g -o cello cello.c
        
clean:
        rm cello

install:
        mkdir -p $(DESTDIR)/usr/bin
        install -m 0755 cello $(DESTDIR)/usr/bin/cello
```

提供一个 LICENSE 文件 

```shell
$ cat LICENSE

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
```

把源代码放到压缩包里

```shell
$ ls  -w1 cello-0.1/
cello.c
LICENSE
Makefile

$ tar -cvzf cello-1.0.tar.gz cello-0.1/
cello-0.1/
cello-0.1/Makefile
cello-0.1/cello.c
cello-0.1/LICENSE
```

把源代码放到 rpm 编译环境

```shell
mv cello-0.1.tar.gz ~/rpmbuild/SOURCES
```

### 编译无 spec 文件的源代码为 rpm 包

#### 使用 `rpmdev-newspec` 创建 `spec` 模板文件 

```shell
$ rpmdev-newspec -o rpmbuild/SPECS/cello.spec
rpmbuild/SPECS/cello.spec created; type minimal, rpm version >= 4.11.

$ cat rpmbuild/SPECS/cello.spec
Name:           cello
Version:
Release:        1%{?dist}
Summary:

License:
URL:
Source0:

BuildRequires:
Requires:

%description


%prep
%setup -q


%build
%configure
make %{?_smp_mflags}


%install
rm -rf $RPM_BUILD_ROOT
%make_install


%files
%doc



%changelog
```

#### 完善配置文件

自行填充上述模板文件的空白内容，如下：

```shell
Name:           cello
Version:        0.1
Release:        1%{?dist}
Summary:       Hello World example implemented in C programming language

License:       GPLv3+
URL:            https://example.com/%{name}
Source0:       https://example.com/%{name}/release/%{name}-%{version}.tar.gz

BuildRequires:  gcc, make

%description
The long-tail description for our Hello World Example implemented in
C programming language.

%prep
%setup -q


%build
make %{?_smp_mflags}


%install
rm -rf $RPM_BUILD_ROOT
%make_install


%files
%doc
%license LICENSE
%{_bindir}/%{name}


%changelog
* Thu Feb 23 2023 Steven Zhao <zhaoweiguo@lczq.com> - 0.1-1
- First bello package
- Example second item in the changelog for version-release 0.1-1
```

#### 开始编译

```shell
$ rpmbuild -bb rpmbuild/SPECS/cello.spec
执行(%prep): /bin/sh -e /var/tmp/rpm-tmp.fbkOro
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ rm -rf cello-0.1
+ /usr/bin/gzip -dc /home/gitlab-runner/rpmbuild/SOURCES/cello-0.1.tar.gz
+ /usr/bin/tar -xf -
+ STATUS=0
+ '[' 0 -ne 0 ']'
+ cd cello-0.1
+ /usr/bin/chmod -Rf a+rX,u+w,g-w,o-w .
+ exit 0
执行(%build): /bin/sh -e /var/tmp/rpm-tmp.s3374A
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd cello-0.1
+ make -j6
gcc -g -o cello cello.c
+ exit 0
执行(%install): /bin/sh -e /var/tmp/rpm-tmp.j5LTBO
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ '[' /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64 '!=' / ']'
+ rm -rf /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
++ dirname /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ mkdir -p /home/gitlab-runner/rpmbuild/BUILDROOT
+ mkdir /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ cd cello-0.1
+ rm -rf /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ /usr/bin/make install DESTDIR=/home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
mkdir -p /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/bin
install -m 0755 cello /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/bin/cello
+ /usr/lib/rpm/find-debuginfo.sh --strict-build-id -m --run-dwz --dwz-low-mem-die-limit 10000000 --dwz-max-die-limit 110000000 /home/gitlab-runner/rpmbuild/BUILD/cello-0.1
extracting debug info from /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/bin/cello
dwz: Too few files for multifile optimization
/usr/lib/rpm/sepdebugcrcfix: Updated 1 CRC32s, 0 CRC32s did match.
1 block
+ /usr/lib/rpm/check-buildroot
+ /usr/lib/rpm/redhat/brp-compress
+ /usr/lib/rpm/redhat/brp-strip-static-archive /usr/bin/strip
+ /usr/lib/rpm/brp-python-bytecompile /usr/bin/python 1
+ /usr/lib/rpm/redhat/brp-python-hardlink
+ /usr/lib/rpm/redhat/brp-java-repack-jars
处理文件：cello-0.1-1.el7.centos.x86_64
执行(%license): /bin/sh -e /var/tmp/rpm-tmp.8tkaP3
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd cello-0.1
+ LICENSEDIR=/home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/share/licenses/cello-0.1
+ export LICENSEDIR
+ /usr/bin/mkdir -p /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/share/licenses/cello-0.1
+ cp -pr LICENSE /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/share/licenses/cello-0.1
+ exit 0
Provides: cello = 0.1-1.el7.centos cello(x86-64) = 0.1-1.el7.centos
Requires(rpmlib): rpmlib(CompressedFileNames) <= 3.0.4-1 rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1
Requires: libc.so.6()(64bit) libc.so.6(GLIBC_2.2.5)(64bit) rtld(GNU_HASH)
处理文件：cello-debuginfo-0.1-1.el7.centos.x86_64
Provides: cello-debuginfo = 0.1-1.el7.centos cello-debuginfo(x86-64) = 0.1-1.el7.centos
Requires(rpmlib): rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1 rpmlib(CompressedFileNames) <= 3.0.4-1
检查未打包文件：/usr/lib/rpm/check-files /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
写道:/home/gitlab-runner/rpmbuild/RPMS/x86_64/cello-0.1-1.el7.centos.x86_64.rpm
写道:/home/gitlab-runner/rpmbuild/RPMS/x86_64/cello-debuginfo-0.1-1.el7.centos.x86_64.rpm
执行(%clean): /bin/sh -e /var/tmp/rpm-tmp.17LUhO
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd cello-0.1
+ /usr/bin/rm -rf /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ exit 0
```

查看文件 

```shell
$ ls  -w1 rpmbuild/RPMS/x86_64/cello-0.1-1.el7.centos.x86_64.rpm
rpmbuild/RPMS/x86_64/cello-0.1-1.el7.centos.x86_64.rpm

$ rpm -qpi rpmbuild/RPMS/x86_64/cello-0.1-1.el7.centos.x86_64.rpm
Name        : cello
Version     : 0.1
Release     : 1.el7.centos
Architecture: x86_64
Install Date: (not installed)
Group       : Unspecified
Size        : 7820
License     : GPLv3+
Signature   : (none)
Source RPM  : cello-0.1-1.el7.centos.src.rpm
Build Date  : 2023年02月23日 星期四 16时40分15秒
Build Host  : backend
Relocations : (not relocatable)
URL         : https://example.com/cello
Summary     : Hello World example implemented in C programming language
Description :
The long-tail description for our Hello World Example implemented in
C programming language.

$ rpm -qpl rpmbuild/RPMS/x86_64/cello-0.1-1.el7.centos.x86_64.rpm
/usr/bin/cello
/usr/share/licenses/cello-0.1
/usr/share/licenses/cello-0.1/LICENSE
```

#### 生成 SRPM 包

```shell
$ rpmbuild -bs rpmbuild/SPECS/cello.spec
写道:/home/gitlab-runner/rpmbuild/SRPMS/cello-0.1-1.el7.centos.src.rpm
```

#### 生成带 spec 文件的源代码压缩包

### 编译自带 spec 文件的源代码为 RPM包

如果你下载的源代码里包含了 `spec` 文件，比如下面这样

```shell
$ tar -tzf cello-0.1.tar.gz
cello-0.1/
cello-0.1/Makefile
cello-0.1/cello.c
cello-0.1/LICENSE
cello-0.1/cello.spec
```

那么可以直接使用 `rpmbuild` 来编译

```shell
$ rpmbuild -tb cello-0.1.tar.gz
执行(%prep): /bin/sh -e /var/tmp/rpm-tmp.5SLEw2
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ rm -rf cello-0.1
+ /usr/bin/gzip -dc /var/tmp/cello-0.1.tar.gz
+ /usr/bin/tar -xf -
+ STATUS=0
+ '[' 0 -ne 0 ']'
+ cd cello-0.1
+ /usr/bin/chmod -Rf a+rX,u+w,g-w,o-w .
+ exit 0
执行(%build): /bin/sh -e /var/tmp/rpm-tmp.tEJFy9
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd cello-0.1
+ make -j6
gcc -g -o cello cello.c
+ exit 0
执行(%install): /bin/sh -e /var/tmp/rpm-tmp.vCKRKg
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ '[' /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64 '!=' / ']'
+ rm -rf /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
++ dirname /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ mkdir -p /home/gitlab-runner/rpmbuild/BUILDROOT
+ mkdir /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ cd cello-0.1
+ rm -rf /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ /usr/bin/make install DESTDIR=/home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
mkdir -p /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/bin
install -m 0755 cello /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/bin/cello
+ /usr/lib/rpm/find-debuginfo.sh --strict-build-id -m --run-dwz --dwz-low-mem-die-limit 10000000 --dwz-max-die-limit 110000000 /home/gitlab-runner/rpmbuild/BUILD/cello-0.1
extracting debug info from /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/bin/cello
dwz: Too few files for multifile optimization
/usr/lib/rpm/sepdebugcrcfix: Updated 1 CRC32s, 0 CRC32s did match.
1 block
+ /usr/lib/rpm/check-buildroot
+ /usr/lib/rpm/redhat/brp-compress
+ /usr/lib/rpm/redhat/brp-strip-static-archive /usr/bin/strip
+ /usr/lib/rpm/brp-python-bytecompile /usr/bin/python 1
+ /usr/lib/rpm/redhat/brp-python-hardlink
+ /usr/lib/rpm/redhat/brp-java-repack-jars
处理文件：cello-0.1-1.el7.centos.x86_64
执行(%license): /bin/sh -e /var/tmp/rpm-tmp.bGZKPo
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd cello-0.1
+ LICENSEDIR=/home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/share/licenses/cello-0.1
+ export LICENSEDIR
+ /usr/bin/mkdir -p /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/share/licenses/cello-0.1
+ cp -pr LICENSE /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/share/licenses/cello-0.1
+ exit 0
Provides: cello = 0.1-1.el7.centos cello(x86-64) = 0.1-1.el7.centos
Requires(rpmlib): rpmlib(CompressedFileNames) <= 3.0.4-1 rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1
Requires: libc.so.6()(64bit) libc.so.6(GLIBC_2.2.5)(64bit) rtld(GNU_HASH)
处理文件：cello-debuginfo-0.1-1.el7.centos.x86_64
Provides: cello-debuginfo = 0.1-1.el7.centos cello-debuginfo(x86-64) = 0.1-1.el7.centos
Requires(rpmlib): rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1 rpmlib(CompressedFileNames) <= 3.0.4-1
检查未打包文件：/usr/lib/rpm/check-files /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
写道:/home/gitlab-runner/rpmbuild/RPMS/x86_64/cello-0.1-1.el7.centos.x86_64.rpm
写道:/home/gitlab-runner/rpmbuild/RPMS/x86_64/cello-debuginfo-0.1-1.el7.centos.x86_64.rpm
执行(%clean): /bin/sh -e /var/tmp/rpm-tmp.jWf8rN
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd cello-0.1
+ /usr/bin/rm -rf /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ exit 0
```

### 编译 SRPM 包到二进制 RPM包

```shell
$ rpmbuild --rebuild rpmbuild/SRPMS/cello-0.1-1.el7.centos.src.rpm
正在安装 rpmbuild/SRPMS/cello-0.1-1.el7.centos.src.rpm
执行(%prep): /bin/sh -e /var/tmp/rpm-tmp.lAlqPt
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ rm -rf cello-0.1
+ /usr/bin/gzip -dc /home/gitlab-runner/rpmbuild/SOURCES/cello-0.1.tar.gz
+ /usr/bin/tar -xf -
+ STATUS=0
+ '[' 0 -ne 0 ']'
+ cd cello-0.1
+ /usr/bin/chmod -Rf a+rX,u+w,g-w,o-w .
+ exit 0
执行(%build): /bin/sh -e /var/tmp/rpm-tmp.5iGy8W
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd cello-0.1
+ make -j6
gcc -g -o cello cello.c
+ exit 0
执行(%install): /bin/sh -e /var/tmp/rpm-tmp.dla0zq
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ '[' /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64 '!=' / ']'
+ rm -rf /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
++ dirname /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ mkdir -p /home/gitlab-runner/rpmbuild/BUILDROOT
+ mkdir /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ cd cello-0.1
+ rm -rf /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ /usr/bin/make install DESTDIR=/home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
mkdir -p /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/bin
install -m 0755 cello /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/bin/cello
+ /usr/lib/rpm/find-debuginfo.sh --strict-build-id -m --run-dwz --dwz-low-mem-die-limit 10000000 --dwz-max-die-limit 110000000 /home/gitlab-runner/rpmbuild/BUILD/cello-0.1
extracting debug info from /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/bin/cello
dwz: Too few files for multifile optimization
/usr/lib/rpm/sepdebugcrcfix: Updated 1 CRC32s, 0 CRC32s did match.
1 block
+ /usr/lib/rpm/check-buildroot
+ /usr/lib/rpm/redhat/brp-compress
+ /usr/lib/rpm/redhat/brp-strip-static-archive /usr/bin/strip
+ /usr/lib/rpm/brp-python-bytecompile /usr/bin/python 1
+ /usr/lib/rpm/redhat/brp-python-hardlink
+ /usr/lib/rpm/redhat/brp-java-repack-jars
处理文件：cello-0.1-1.el7.centos.x86_64
执行(%license): /bin/sh -e /var/tmp/rpm-tmp.xOCrFU
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd cello-0.1
+ LICENSEDIR=/home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/share/licenses/cello-0.1
+ export LICENSEDIR
+ /usr/bin/mkdir -p /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/share/licenses/cello-0.1
+ cp -pr LICENSE /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64/usr/share/licenses/cello-0.1
+ exit 0
Provides: cello = 0.1-1.el7.centos cello(x86-64) = 0.1-1.el7.centos
Requires(rpmlib): rpmlib(CompressedFileNames) <= 3.0.4-1 rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1
Requires: libc.so.6()(64bit) libc.so.6(GLIBC_2.2.5)(64bit) rtld(GNU_HASH)
处理文件：cello-debuginfo-0.1-1.el7.centos.x86_64
Provides: cello-debuginfo = 0.1-1.el7.centos cello-debuginfo(x86-64) = 0.1-1.el7.centos
Requires(rpmlib): rpmlib(FileDigests) <= 4.6.0-1 rpmlib(PayloadFilesHavePrefix) <= 4.0-1 rpmlib(CompressedFileNames) <= 3.0.4-1
检查未打包文件：/usr/lib/rpm/check-files /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
写道:/home/gitlab-runner/rpmbuild/RPMS/x86_64/cello-0.1-1.el7.centos.x86_64.rpm
写道:/home/gitlab-runner/rpmbuild/RPMS/x86_64/cello-debuginfo-0.1-1.el7.centos.x86_64.rpm
执行(%clean): /bin/sh -e /var/tmp/rpm-tmp.HpZMfn
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ cd cello-0.1
+ /usr/bin/rm -rf /home/gitlab-runner/rpmbuild/BUILDROOT/cello-0.1-1.el7.centos.x86_64
+ exit 0
执行(--clean): /bin/sh -e /var/tmp/rpm-tmp.vhEPtR
+ umask 022
+ cd /home/gitlab-runner/rpmbuild/BUILD
+ rm -rf cello-0.1
+ exit 0
```


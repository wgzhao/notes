# 快速创建一个虚假 RPM 包

有的时候有一些包有依赖，但是依赖的文件可能永远用不到，因此我们可以采取快速创建一个虚拟 RPM 包的方式“骗过”系统的依赖检查，以下是快速创建 RPM 包的脚本

```shell
#!/bin/sh
# This scripts creates and build a simple RPM package
#
# Prerequisites:
#  - rpm-build, make and gcc (as it's a c file) packages must be installed
#

# Holds the name of the root directory containing the necessary structure to
# build RPM packages.
[ $UID -eq 0 ] || exit 2
RPM_ROOT_DIR=~/rpm_factory
PKG_NAME=${1-"dummy_package"}
PKG_TAR=/tmp/${PKG_NAME}.tar.gz
BINARY_FILE=hello_world
# Recreate the root directory and its structure if necessary
mkdir -p ${RPM_ROOT_DIR}/{SOURCES,BUILD,RPMS,SPECS,SRPMS,tmp}
PKG_DIR=/tmp/${PKG_NAME}
mkdir -p ${PKG_DIR}

cat << __EOF__ > ${PKG_DIR}/hello_world.c
#include <stdio.h>
int main(){
  printf("Hello world!\n");
  return 0;
}
__EOF__

pushd ${PKG_DIR}/..
tar czvf ${PKG_NAME}.tar.gz ${PKG_NAME}
popd
pushd  $RPM_ROOT_DIR
cp ${PKG_TAR} ${RPM_ROOT_DIR}/SOURCES/

# Creating a basic spec file
cat << __EOF__ > ${RPM_ROOT_DIR}/SPECS/${PKG_NAME}.spec
Summary: This package is a sample for quickly build dummy RPM package.
Name: $PKG_NAME
Version: ${2-:1.0}
Release: ${3-:0}
License: GPL
Packager: $USER
Group: Development/Tools
Source: %{name}.tar.gz
BuildRequires: coreutils
BuildRoot: ${RPM_ROOT_DIR}/tmp/%{name}-%{version}

%description
%{summary}

%prep
%setup -n ${PKG_NAME}

%build
make $BINARY_FILE

%install
mkdir -p "%{buildroot}/opt/${PKG_NAME}"
cp $BINARY_FILE "%{buildroot}/opt/${PKG_NAME}/"

%files
/opt/${PKG_NAME}/hello_world

%clean
%if "%{clean}" != ""
  rm -rf %{_topdir}/BUILD/%{name}
  [ $(basename %{buildroot}) == "%{name}-%{version}-%{release}.%{_target_cpu}" ] && rm -rf %{buildroot}
%endif

%post
chmod 755 -R /opt/${PKG_NAME}
__EOF__

rpmbuild -v -bb --define "_topdir ${RPM_ROOT_DIR}" SPECS/${PKG_NAME}.spec
popd
```

用法如下

```shell
create_dummy_rpm.sh <package_name> <version> <release-version>
```

比如

```shell
create_dummy_rpm.sh texinfo-tex 5.1 4.el7
```

则生成的 rpm 文件为 `/root/rpm_factory/RPM/x86_64/texinfo-text-5.1-4.el7.x86_64.rpm`

---
title: 安装impala-shell遇到的问题
date: 2017-12-05
tags: 
  - impala
  - 问题排查
---
### 下载rpm包

 [impala-shell-2.2.0+cdh5.4.9+0-1.cdh5.4.9.p0.30.el6.x86_64.rpm](http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/5.4.9/RPMS/x86_64/)

### 安装

```
# yum install python-setuptools
# rpm -ivh impala-shell-2.2.0+cdh5.4.9+0-1.cdh5.4.9.p0.30.el6.x86_64.rpm
```

#### 报错

```
error: Failed dependencies: libpython2.6.so.1.0()(64bit) is needed by impala-shell-2.2.0+cdh5.4.9+0-1.cdh5.4.9.p0.30.el6.x86_64.rpm
```

#### 下载并安装

```
rpm -Uvh lib64python2.6-2.6.6-1mdv2011.0.x86_64.rpm
```

### 连接

```
# impala-shell
```

#### 报错

```
···
···
ImportError: libsasl2.so.2: cannot open shared object file: No such file or directory
```

#### 检查sasl

```
rpm -qa | grep sasl
rpm -ql cyrus-sasl-lib
```

发现 libsasl2.so.3被 cyrus-sasl-lib 安装在了 /usr/lib64/.这个目录下

#### 重装sasl

```
pip uninstall sasl
pip install sasl
```

pip不存在

#### 安装pip

```
yun install -y pip
Nothing to do
```

#### 安装epel扩展源

```
sudo yum -y install epel-release
yum -y install python-pip
```

#### 卸载sasl

```
 pip uninstall sasl
```

无法删除

```
Cannot uninstall requirement sasl, not installed
```

#### 安装sasl

```
pip install sasl

sasl/saslwrapper.cpp:8:22: fatal error: pyconfig.h: No such file or directory
#include "pyconfig.h"
^
compilation terminated.
error: command 'gcc' failed with exit status 1
```

#### 继续找问题，回到这一步

```
ImportError: libsasl2.so.2: cannot open shared object file: No such file or directory
```

#### 解决方案

```
Sometimes these version problems can be resolved by creating a symbolic link with another version of the same library. If you have libsasl2.so installed, create a linked file libsasl2.so.2 for the same:

sudo ln -s /usr/lib64/libsasl2.so /usr/lib64/libsasl2.so.2
```
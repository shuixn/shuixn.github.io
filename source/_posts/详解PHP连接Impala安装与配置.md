---
title: 详解PHP连接Impala安装与配置
date: 2017-12-08
categories:
  - 技术
tags: 
  - PHP 
  - impala
---
### Impala的SQL语法参考

https://www.cloudera.com/documentation/enterprise/latest/topics/impala_langref_sql.html

<!-- more -->

### PHP通过Thrift连接Impala

#### 安装thrift服务

- Thrift最初由Facebook开发用做系统内各语言之间的RPC通信.
- Thrift是一款可伸缩跨语言的服务开发框架, 该框架已经开源并且加入的Apache项目.
- Thrift主要功能是: 通过自定义的Interface Definition Language(IDL), 可以创建基于RPC的客户端和服务端的服务代码.
- 数据和服务代码的生成是通过Thrift内置的代码生成器来实现的, Thrift的跨语言性体现在:它可以生成C++/Java/Python/PHP/Ruby/Erlang/Perl/Haskell/C#/Cocoa/JavaScript/Node.js/Smalltalk/OCaml/Delphi等语言的代码,且它们之间可以进行透明的通信.

```bash
[root@localhost tools]# rpm -ivh thrift-0.9.0-28.1.i686.rpm

warning: thrift-0.9.0-28.1.i686.rpm: Header V3 DSA/SHA1 Signature, key ID a949b429: NOKEY
error: Failed dependencies:
libc.so.6 is needed by thrift-0.9.0-28.1.i686
libc.so.6(GLIBC_2.0) is needed by thrift-0.9.0-28.1.i686
libc.so.6(GLIBC_2.1) is needed by thrift-0.9.0-28.1.i686
libc.so.6(GLIBC_2.1.3) is needed by thrift-0.9.0-28.1.i686
libc.so.6(GLIBC_2.3) is needed by thrift-0.9.0-28.1.i686
libc.so.6(GLIBC_2.3.4) is needed by thrift-0.9.0-28.1.i686
libc.so.6(GLIBC_2.4) is needed by thrift-0.9.0-28.1.i686
libgcc_s.so.1 is needed by thrift-0.9.0-28.1.i686
libgcc_s.so.1(GCC_3.0) is needed by thrift-0.9.0-28.1.i686
libm.so.6 is needed by thrift-0.9.0-28.1.i686
libstdc++.so.6 is needed by thrift-0.9.0-28.1.i686
libstdc++.so.6(CXXABI_1.3) is needed by thrift-0.9.0-28.1.i686
libstdc++.so.6(CXXABI_1.3.1) is needed by thrift-0.9.0-28.1.i686
libstdc++.so.6(GLIBCXX_3.4) is needed by thrift-0.9.0-28.1.i686
libstdc++.so.6(GLIBCXX_3.4.11) is needed by thrift-0.9.0-28.1.i686
libstdc++.so.6(GLIBCXX_3.4.9) is needed by thrift-0.9.0-28.1.i686

#报错, 缺少相关依赖.
#解决libc.so.6依赖:

[root@localhost tools]# yum list glibc*
[root@localhost tools]# yum install glibc.i686

#解决libgcc_s.so.1依赖:到https://rpmfind.net/linux/rpm2html/search.php?query=libgcc_s.so.1下载相关依赖rpm包

[root@localhost tools]# rpm -ivh libgcc-4.4.7-18.el6.x86_64.rpm
[root@localhost tools]# rpm -ivh libgcc-4.4.7-18.el6.i686.rpm

#解决libstdc++.so.6依赖:
#到https://rpmfind.net/linux/rpm2html/search.php?query=libstdc++.so.6下载相关依赖rpm包

[root@localhost tools]# rpm -ivh libstdc++-4.4.7-18.el6.i686.rpm

#最后重新安装即可:
[root@localhost tools]# rpm -ivh thrift-0.9.0-28.1.i686.rpm
        warning: thrift-0.9.0-28.1.i686.rpm: Header V3 DSA/SHA1 Signature, key ID a949b429: NOKEY
        Preparing...                ########################################### [100%]
           1:thrift                 ########################################### [100%]
```

#### 检查是否安装成功

a. 首先创建Thrift的语法规则文件, 命名为server.thrift, 内容如下:

```vim
struct message
{
    i32 seqId,
    string content
}
service serDemo
{
    void put(message msg)
}
```

b. 然后shell下执行命令:

```bash
# thrift -gen php server.thrift
```

该语句用于创建php服务框架, 创建成功后会在该目录下生成gen-php文件夹.

#### impala的SQL查询参考

http://www.cloudera.com/documentation/archive/impala/2-x/2-1-x/topics/impala_select.html

#### 连接PHP例子参考

https://github.com/Automattic/php-thrift-sql

```bash
# git clone https://github.com/Automattic/php-thrift-sql.git
# cd php-thrift-sql/
# php -c php.ini build.php  【重新生成ThriftSQL.phar文件】
# vim test.php  
```

内容如下:

```php
<?php
// Load this lib
require_once __DIR__ . '/ThriftSQL.phar';
// 1.Try out a Hive query
$hive = new \ThriftSQL\Hive('192.168.8.207', 21050);
$hiveTables = $hive
    ->setSasl(false) // To turn SASL auth off, on by default
    ->connect()
    ->queryAndFetchAll('SHOW TABLES');
print_r($hiveTables);
// 2.Try out an Impala query
$impala = new \ThriftSQL\Impala('192.168.8.207');
$impalaTables = $impala
    ->connect()
    ->queryAndFetchAll('SHOW TABLES');
print_r($impalaTables);
// 3.Try out an Impala query
$impalaDatas = $impala
    ->connect()
    ->queryAndFetchAll('select * from db_mcfx_log.t_log_sdk_log_user');
print_r($impalaDatas);
// 4.Don't forget to clear the client and close socket.
$hive->disconnect();
$impala->disconnect();
?>
```

然后执行test.php

```bash
# php test.php
```

### 使PHP通过ODBC连接Impala(方法1)

#### 安装ODBC相关软件包以及ImpalaODBC驱动

##### 系统版本

centos6，别用centos7，很多lib不兼容，经常会看到这样的错误，解决起来非常棘手，即使下载安装了i686相关的包也不行。

```bash
libxxx()(64bit) is needed by xxx
```

#### 使用ODBC连接数据库, 分为两个主要部分:

- 安装ODBC管理程序(例如unixODBC/iODBC等), 这里管理程序选择unixODBC
- 安装每个数据库对应的ODBC驱动程序, 而Impala的ODBC驱动则参考Cloudera-ODBC-Driver-for-Impala-Install-Guide

#### ClouderaImpalaODBC到Cloudera官网下载:

https://downloads.cloudera.com/connectors/impala-2.5.15.1015/Linux/EL6/ClouderaImpalaODBC-2.5.15.1015-1.el6.x86_64.rpm

#### 若已安装其他版本则先卸载原来安装的

```bash
# rpm -q ClouderaImpalaODBC
# rpm -e ClouderaImpalaODBC-2.5.30.1011-1.x86_64
# yum install -y unixODBC*
# rpm -ivh ClouderaImpalaODBC-2.5.15.1015-1.el6.x86_64.rpm
```

使用odbcinst命令查看unixODBC配置文件路径.
不同版本的unixODBC配置文件路径是不同的, 如果是源代码方式安装unixODBC, 也可以通过编译参数--sysconfdir指定.

```bash
[root@localhost ~]# odbcinst -j
unixODBC 2.2.14
DRIVERS............: /etc/odbcinst.ini
SYSTEM DATA SOURCES: /etc/odbc.ini
FILE DATA SOURCES..: /etc/ODBCDataSources
USER DATA SOURCES..: /root/.odbc.ini
SQLULEN Size.......: 8
SQLLEN Size........: 8
SQLSETPOSIROW Size.: 8
```

#### 增加环境变量(注意重启后失效, 可在/etc/profile里面改使永久生效)

```bash
# export LD_LIBRARY_PATH=/usr/local/lib:/opt/cloudera/impalaodbc/lib/64
```

#### 动态编译pdo-odbc扩展

```bash
[root@localhost ~]# cd /home/xxx/php-7.1.3 # 根据自己的PHP源码路径
[root@localhost php-7.1.3]# cd ext/pdo_odbc/
[root@localhost pdo_odbc]# /application/php/bin/phpize
[root@localhost pdo_odbc]# ./configure --help 
# 注意，PHP多版本共存下需要指定PHP
[root@localhost pdo_odbc]# ./configure --with-php-config=/xxx/bin/php-config --with-pdo-odbc=unixODBC,/usr/

# configure: error: Cannot find header file(s) for pdo_odbc
# 完全安装好unixODBC和unixODBC-devel

[root@localhost pdo_odbc]# make && make install
# 编辑php.ini加入pdo_odbc.so扩展
[root@localhost pdo_odbc]# vim /xxx/php.ini
extension = pdo_odbc.so

# 重启php查看配置如下:
[root@localhost pdo_odbc]# /etc/init.d/php-fpm restart
[root@localhost pdo_odbc]# php -i |grep odbc
PDO drivers => mysql, sqlite, odbc
LD_LIBRARY_PATH => /usr/local/lib:/opt/cloudera/impalaodbc/lib/64
$_SERVER['LD_LIBRARY_PATH'] => /usr/local/lib:/opt/cloudera/impalaodbc/lib/64
```

#### 动态编译odbc扩展

```bash
[root@localhost ~]# cd /home/xxx/php-7.1.3
[root@localhost php-7.1.3]# cd ext/odbc/
[root@localhost odbc]# /application/php/bin/phpize
[root@localhost odbc]# ./configure --help
[root@localhost odbc]# ./configure --with-php-config=/xxx/php7.1.3/bin/php-config --with-unixODBC=/usr/
[root@localhost odbc]# make
[root@localhost odbc]# make install
Installing shared extensions:     /xxx/php7.1.3/lib/php/extensions/no-debug-non-zts-20160303/
root@localhost odbc]# ll /xxx/php7.1.3/lib/php/extensions/no-debug-non-zts-20160303/
total 5556
-rwxr-xr-x 1 root root  350680 Mar 24 15:42 memcached.so
-rwxr-xr-x 1 root root  255310 Mar 24 15:00 memcache.so
-rwxr-xr-x 1 root root  173215 Jun 12 09:24 odbc.so
-rwxr-xr-x 1 root root 3020972 Mar 24 14:23 opcache.a
-rwxr-xr-x 1 root root 1750437 Mar 24 14:23 opcache.so
-rwxr-xr-x 1 root root  126680 May 23 14:04 pdo_odbc.so
# 编辑php.ini加入odbc.so扩展
[root@localhost odbc]# vim /xxx/php/lib/php.ini
extension = odbc.so
# 重启php查看配置信息:
[root@localhost odbc]# /etc/init.d/php-fpm restart
[root@localhost odbc]# php -i |grep odbc
odbc
odbc.allow_persistent => On => On
odbc.check_persistent => On => On
odbc.default_cursortype => Static cursor => Static cursor
odbc.default_db => no value => no value
odbc.default_pw => no value => no value
odbc.default_user => no value => no value
odbc.defaultbinmode => return as is => return as is
odbc.defaultlrl => return up to 4096 bytes => return up to 4096 bytes
odbc.max_links => Unlimited => Unlimited
odbc.max_persistent => Unlimited => Unlimited
PDO drivers => mysql, sqlite, odbc
LD_LIBRARY_PATH => /usr/local/lib:/opt/cloudera/impalaodbc/lib/64
PWD => /xxx/php-7.1.3/ext/odbc
$SERVER['LD_LIBRARY_PATH'] => /usr/local/lib:/opt/cloudera/impalaodbc/lib/64
$_SERVER['PWD'] => /xxx/php-7.1.3/ext/odbc

# 注意, 如果configure编译的时候出现报错如下:
checking for Adabas support... cp: cannot stat '/usr/local/lib/odbclib.a': No such file or directory
configure: error: ODBC header file '/usr/local/incl/sqlext.h' not found!

# 那么可进行修改configure配置文件:
[root@localhost odbc]# sed -ri 's@^ *test +"\$PHP." *= *"no" *&& *PHP_.=yes *$@#&@g' configure
```

#### 修改/etc/odbc.ini替换为如下:(注意！！下文所有的配置文件的每行都一定要保持左对齐！)

```vim
[ODBC]

Specify any global ODBC configuration here such as ODBC tracing.
[ODBC Data Sources]
#Cloudera Hive 32-bit=Cloudera ODBC Driver for Apache Hive 32-bit
impala=Cloudera ODBC Driver for Impala 64-bit
[impala]

Description: DSN Description.
This key is not necessary and is only to give a description of the data source.
Description=Cloudera ODBC Driver for Impala (64-bit) DSN

Driver: The location where the ODBC driver is installed to.
Driver=/opt/cloudera/impalaodbc/lib/64/libclouderaimpalaodbc64.so

The DriverUnicodeEncoding setting is only used for SimbaDM
When set to 1, SimbaDM runs in UTF-16 mode.
When set to 2, SimbaDM runs in UTF-8 mode.
#DriverUnicodeEncoding=2

Values for HOST, PORT, KrbFQDN, and KrbServiceName should be set here.
They can also be specified on the connection string.
HOST=xxx
PORT=xxx
Database=default

The authentication mechanism.
0 - No authentication (NOSASL)
1 - Kerberos authentication (SASL)
2 - Username authentication (SASL)
3 - Username/password authentication (NOSASL or SASL depending on UseSASL configuration)
AuthMech=0

Set to 1 to use SASL for authentication.
Set to 0 to not use SASL.
When using Kerberos authentication (SASL) or Username authentication (SASL) SASL is always used
and this configuration is ignored. SASL is always not used for No authentication (NOSASL).
UseSASL=0

Kerberos related settings.
KrbFQDN=
KrbRealm=
KrbServiceName=

Username/password authentication with SASL settings.
UID=hdfs
PWD=

Set to 0 to disable SSL.
Set to 1 to enable SSL.
SSL=0
CAIssuedCertNamesMismatch=1
TrustedCerts=/opt/cloudera/impalaodbc/lib/64/cacerts.pem

General settings
TSaslTransportBufSize=1000
RowsFetchedPerBlock=10000
SocketTimeout=0
StringColumnLength=32767
UseNativeQuery=0
```

#### 修改/etc/odbcins.ini替换为如下:

```vim
[ODBC Drivers]
Cloudera ODBC Driver for Impala 64-bit=Installed
#[Cloudera ODBC Driver for Apache Hive 32-bit]
#Description=Cloudera ODBC Driver for Apache Hive (32-bit)
#Driver=/opt/cloudera/hiveodbc/lib/32/libclouderahiveodbc32.so

The option below is for using unixODBC when compiled with -DSQL_WCHART_CONVERT.
Execute 'odbc_config --cflags' to determine if you need to uncomment it.
IconvEncoding=UCS-4LE
[Cloudera ODBC Driver for Impala 64-bit]
Description=Cloudera ODBC Driver for Impala (64-bit)
Driver=/opt/cloudera/impalaodbc/lib/64/libclouderaimpalaodbc64.so

The option below is for using unixODBC when compiled with -DSQL_WCHART_CONVERT.
Execute 'odbc_config --cflags' to determine if you need to uncomment it.
IconvEncoding=UCS-4LE
```

#### 修改/etc/cloudera.impalaodbc.ini

```vim
[Driver]

- Note that this default DriverManagerEncoding of UTF-32 is for iODBC.
- unixODBC uses UTF-16 by default.
- If unixODBC was compiled with -DSQL_WCHART_CONVERT, then UTF-32 is the correct value.
##   Execute 'odbc_config --cflags' to determine if you need UTF-32 or UTF-16 on unixODBC

- SimbaDM can be used with UTF-8 or UTF-16.
##   The DriverUnicodeEncoding setting will cause SimbaDM to run in UTF-8 when set to 2 or UTF-16 when set to 1.
DriverManagerEncoding=UTF-32
ErrorMessagesPath=/opt/cloudera/impalaodbc/ErrorMessages
LogLevel=0
LogPath=

- Uncomment the ODBCInstLib corresponding to the Driver Manager being used.
- Note that the path to your ODBC Driver Manager must be specified in LD_LIBRARY_PATH (LIBPATH for AIX).
- Note that AIX has a different format for specifying its shared libraries.
Generic ODBCInstLib
#   iODBC
#ODBCInstLib=libiodbcinst.so
#   SimbaDM / unixODBC
ODBCInstLib=libodbcinst.so

AIX specific ODBCInstLib
#   iODBC
#ODBCInstLib=libiodbcinst.a(libiodbcinst.so.2)
#   SimbaDM
#ODBCInstLib=libodbcinst.a(odbcinst.so)
#   unixODBC
#ODBCInstLib=libodbcinst.a(libodbcinst.so.1)
```

#### 测试连接impala

因为在/etc/odbc.ini我们设定的DSN名称为impala, 所以执行如下语句:

```bash
[root@localhost ~]# isql -v 'impala'
        +---------------------------------------+
        | Connected!                            |
        |                                       |
        | sql-statement                         |
        | help [tablename]                      |
        | quit                                  |
        |                                       |
        +---------------------------------------+

```

### PHP通过ODBC连接Impala(方法2)

#### 安装ODBC相关软件包以及ImpalaODBC驱动

```bash
# yum install -y unixODBC*
# rpm -ivh ClouderaImpalaODBC-2.5.15.1015-1.el6.x86_64.rpm
```

#### 新建存放配置的目录

```bash
[root@localhost ~]# mkdir /home/xxx/config_odbc
[root@localhost ~]# cd /home/xxx/config_odbc
[root@localhost config_odbc]# mkdir odbc
```

#### 创建环境变量文件common.env

```vim
[root@localhost config_odbc]# vim common.env

CONFIG_DIR=$(readlink -f $(dirname ${BASH_SOURCE[0]}))
ODBC_CONFIG_DIR=${CONFIG_DIR}/odbc
export ODBCINI=${ODBC_CONFIG_DIR}/odbc.ini
export ODBCSYSINI=${ODBC_CONFIG_DIR}
export CLOUDERAIMPALAINI=${ODBC_CONFIG_DIR}/cloudera.impalaodbc.ini
export CLOUDERAHIVEINI=${ODBC_CONFIG_DIR}/cloudera.hiveodbc.ini
export LD_LIBRARY_PATH=/usr/local/lib:/opt/cloudera/impalaodbc/lib/64
```

#### 创建配置odbc/odbc.ini

```vim
[root@localhost config_odbc]# vim odbc/odbc.ini    【注意里面包含了ODBC连接hive和impala的配置】

[ODBC]
# Specify any global ODBC configuration here such as ODBC tracing.
[ODBC Data Sources]
#Cloudera Hive 32-bit=Cloudera ODBC Driver for Apache Hive 32-bit
hive=Cloudera ODBC Driver for Apache Hive 64-bit
impala=Cloudera ODBC Driver for Impala 64-bit
[hive]
# Description: DSN Description.
# This key is not necessary and is only to give a description of the data source.
Description=Cloudera ODBC Driver for Apache Hive (64-bit) DSN
# Driver: The location where the ODBC driver is installed to.
Driver=/opt/cloudera/hiveodbc/lib/64/libclouderahiveodbc64.so
# When using No Service Discovery, specify the IP address or host name of the Hive server.
# When using ZooKeeper as the Service Discovery Mode, specify a comma-separated list of ZooKeeper
# servers in the following format:
#       <zk_host1:zk_port1>,<zk_host2:zk_port2>,...
HOST=192.168.11.9
# The TCP port Hive server is listening. This is not required when using ZooKeeper as the service
# discovery mode as the port is specified in the HOST connection attribute.
PORT=10000
# The name of the database schema to use when a schema is not explicitly specified in a query.
Schema=default
# Set to 0 to when connecting directory to Hive Server 2 (No Service Discovery).
# Set to 1 to do Hive Server 2 service discovery using ZooKeeper.
# Note service discovery is not support when using Hive Server 1.
ServiceDiscoveryMode=0
# The namespace on ZooKeeper under which Hive Server 2 znodes are added. Required only when doing
# HS2 service discovery with ZooKeeper (ServiceDiscoveryMode=1).
ZKNamespace=
# Set to 1 if you are connecting to Hive Server 1. Set to 2 if you are connecting to Hive Server 2.
HiveServerType=2
# The authentication mechanism to use for the connection.
#   Set to 0 for No Authentication
#   Set to 1 for Kerberos
#   Set to 2 for User Name
#   Set to 3 for User Name and Password
# Note only No Authentication is supported when connecting to Hive Server 1.
AuthMech=2
# The Thrift transport to use for the connection.
#    Set to 0 for Binary
#    Set to 1 for SASL
#    Set to 2 for HTTP
# Note for Hive Server 1 only Binary can be used.
ThriftTransport=1
# When this option is enabled (1), the driver does not transform the queries emitted by an
# application, so the native query is used.
# When this option is disabled (0), the driver transforms the queries emitted by an application and
# converts them into an equivalent from in HiveQL.
UseNativeQuery=0
# Set the UID with the user name to use to access Hive when using AuthMech 2 to 8.
UID=hdfs
# The following is settings used when using Kerberos authentication (AuthMech 1 and 10)
# The fully qualified host name part of the of the Hive Server 2 Kerberos service principal.
# For example if the service principal name of you Hive Server 2 is:
#   hive/myhs2.mydomain.com@EXAMPLE.COM
# Then set KrbHostFQDN to myhs2.mydomain.com
KrbHostFQDN=_HOST
# The service name part of the of the Hive Server 2 Kerberos service principal.
# For example if the service principal name of you Hive Server 2 is:
#   hive/myhs2.mydomain.com@EXAMPLE.COM
# Then set KrbServiceName to hive
KrbServiceName=hive
# The realm part of the of the Hive Server 2 Kerberos service principal.
# For example if the service principal name of you Hive Server 2 is:
#   hive/myhs2.mydomain.com@EXAMPLE.COM
# Then set KrbRealm to EXAMPLE.COM
KrbRealm=
# Set to 1 to enable SSL. Set to 0 to disable.
SSL=0
# Set to 1 to enable two-way SSL. Set to 0 to disable. You must enable SSL in order to
# use two-way SSL.
TwoWaySSL=0
# The file containing the client certificate in PEM format. This is required when using two-way SSL.
ClientCert=
# The client private key. This is used for two-way SSL authentication.
ClientPrivateKey=
# The password for the client private key. Password is only required for password protected
# client private key.
ClientPrivateKeyPassword=
[impala]
# Description: DSN Description.
# This key is not necessary and is only to give a description of the data source.
Description=Cloudera ODBC Driver for Impala (64-bit) DSN
# Driver: The location where the ODBC driver is installed to.
Driver=/opt/cloudera/impalaodbc/lib/64/libclouderaimpalaodbc64.so
# The DriverUnicodeEncoding setting is only used for SimbaDM
# When set to 1, SimbaDM runs in UTF-16 mode.
# When set to 2, SimbaDM runs in UTF-8 mode.
#DriverUnicodeEncoding=2
# Values for HOST, PORT, KrbFQDN, and KrbServiceName should be set here.
# They can also be specified on the connection string.
HOST=192.168.11.10
PORT=21050
Database=default
# The authentication mechanism.
# 0 - No authentication (NOSASL)
# 1 - Kerberos authentication (SASL)
# 2 - Username authentication (SASL)
# 3 - Username/password authentication (NOSASL or SASL depending on UseSASL configuration)
AuthMech=3
# Set to 1 to use SASL for authentication.
# Set to 0 to not use SASL.
# When using Kerberos authentication (SASL) or Username authentication (SASL) SASL is always used
# and this configuration is ignored. SASL is always not used for No authentication (NOSASL).
UseSASL=0
# Kerberos related settings.
KrbFQDN=
KrbRealm=
KrbServiceName=
# Username/password authentication with SASL settings.
UID=hdfs
PWD=
# Set to 0 to disable SSL.
# Set to 1 to enable SSL.
SSL=0
CAIssuedCertNamesMismatch=1
TrustedCerts=/opt/cloudera/impalaodbc/lib/64/cacerts.pem
# General settings
TSaslTransportBufSize=1000
RowsFetchedPerBlock=10000
SocketTimeout=0
StringColumnLength=32767
UseNativeQuery=0
```

#### 创建配置odbc/odbcinst.ini

```vim
[root@localhost config_odbc]# vim odbc/odbcinst.ini    【注意里面包含了ODBC连接hive和impala的配置】

[ODBC Drivers]
Cloudera ODBC Driver for Apache Hive 64-bit=Installed
Cloudera ODBC Driver for Impala 64-bit=Installed
#[Cloudera ODBC Driver for Apache Hive 32-bit]
#Description=Cloudera ODBC Driver for Apache Hive (32-bit)
#Driver=/opt/cloudera/hiveodbc/lib/32/libclouderahiveodbc32.so
[Cloudera ODBC Driver for Apache Hive 64-bit]
Description=Cloudera ODBC Driver for Apache Hive (64-bit)
Driver=/opt/cloudera/hiveodbc/lib/64/libclouderahiveodbc64.so
## The option below is for using unixODBC when compiled with -DSQL_WCHART_CONVERT.
## Execute 'odbc_config --cflags' to determine if you need to uncomment it.
# IconvEncoding=UCS-4LE
[Cloudera ODBC Driver for Impala 64-bit]
Description=Cloudera ODBC Driver for Impala (64-bit)
Driver=/opt/cloudera/impalaodbc/lib/64/libclouderaimpalaodbc64.so
## The option below is for using unixODBC when compiled with -DSQL_WCHART_CONVERT.
## Execute 'odbc_config --cflags' to determine if you need to uncomment it.
# IconvEncoding=UCS-4LE
```

#### 创建配置odbc/cloudera.impalaodbc.ini

```vim
[root@localhost config_odbc]# vim odbc/cloudera.impalaodbc.ini    【注意里面包含了ODBC连接hive和impala的配置】

[Driver]
## - Note that this default DriverManagerEncoding of UTF-32 is for iODBC.
## - unixODBC uses UTF-16 by default.
## - If unixODBC was compiled with -DSQL_WCHART_CONVERT, then UTF-32 is the correct value.
##   Execute 'odbc_config --cflags' to determine if you need UTF-32 or UTF-16 on unixODBC
## - SimbaDM can be used with UTF-8 or UTF-16.
##   The DriverUnicodeEncoding setting will cause SimbaDM to run in UTF-8 when set to 2 or UTF-16 when set to 1.
DriverManagerEncoding=UTF-32
ErrorMessagesPath=/opt/cloudera/impalaodbc/ErrorMessages
LogLevel=0
LogPath=
## - Uncomment the ODBCInstLib corresponding to the Driver Manager being used.
## - Note that the path to your ODBC Driver Manager must be specified in LD_LIBRARY_PATH (LIBPATH for AIX).
## - Note that AIX has a different format for specifying its shared libraries.
# Generic ODBCInstLib
#   iODBC
#ODBCInstLib=libiodbcinst.so
#   SimbaDM / unixODBC
ODBCInstLib=libodbcinst.so
# AIX specific ODBCInstLib
#   iODBC
#ODBCInstLib=libiodbcinst.a(libiodbcinst.so.2)
#   SimbaDM
#ODBCInstLib=libodbcinst.a(odbcinst.so)
#   unixODBC
#ODBCInstLib=libodbcinst.a(libodbcinst.so.1)
```

#### 创建配置cloudera.hiveodbc.ini

```vim
[root@localhost config_odbc]# vim odbc/cloudera.hiveodbc.ini    【注意里面包含了ODBC连接hive和impala的配置】

[Driver]
ErrorMessagesPath=/opt/cloudera/hiveodbc/ErrorMessages/
LogLevel=0
LogPath=
SwapFilePath=/tmp
```

#### 进行测试

```bash
[root@localhost config_odbc]# source common.env
        [root@localhost config_odbc]# isql -v 'impala'
        +---------------------------------------+
        | Connected!                            |
        |                                       |
        | sql-statement                         |
        | help [tablename]                      |
        | quit                                  |
        |                                       |
        +---------------------------------------+

```

注意前提是, 连接的IP的端口已经开启了此机器的访问权限, 否则会出现如下错误:

```bash
[root@localhost config_odbc]# isql -v 'impala'
        [S1000][unixODBC][Cloudera][ImpalaODBC] (100) Error from the Impala Thrift API: connect() failed: Connection timed out
        [ISQL]ERROR: Could not SQLConnect
```

#### 运行php进行测试

1)如果是安装odbc扩展的, 如下:

```bash
./configure --prefix=$php_root --with-config-file-path=/etc --with-mysql=$mysql_root --with-pdo-mysql=$mysql_root/bin/mysql_config --with-mysqli=$mysql_root/bin/mysql_config --with-iconv-dir=/usr/local --with-freetype-dir --with-jpeg-dir --with-png-dir --enable-gd-native-ttf --enable-zip --with-zlib --with-gd --disable-rpath --enable-bcmath --enable-shmop --enable-sysvsem --with-curl --with-curlwrappers --enable-mbstring --with-mcrypt --disable-ipv6 --enable-static --enable-maintainer-zts --enable-sockets --enable-soap --with-openssl --without-pdo-sqlite --enable-fpm --with-unixODBC=/usr/
```

那么, 就用面向过程的写法:

```php
[root@localhost ~]# vim test-php-odbc-impala.php

<?php
$conn = odbc_connect("impala", "", "");
echo "开始时间:" . time() . "\r\n";
$sql = "select * from db_mcfx_log.t_log_sdk_log_pay";
echo "准备执行sql语句: $sql\r\n";
$rs = odbc_exec($conn, $sql);
echo "语句执行完成, 准备开始获取结果集\r\n";
$result = [];
while ($row=odbc_fetch_array($rs)) {
    $result[] = $row;
}
echo "结果获取完毕\r\n";
echo "总记录数量: " . sizeof($result) . "\r\n";
odbc_close($conn);
var_dump($conn);
echo "结束时间: " . time() . "\r\n";
```

2)如果是安装pdo_odbc扩展的, 那么就用面向过程的方法如下测试:

```php
<?php
$dbh= new PDO('odbc:impala', '', '');
$sql = "select count(*) from db_mcfx_log.t_log_sdk_log_pay";
$stmt = $dbh->prepare("$sql");
$stmt->execute();
while ($row = $stmt->fetch()) {
    print_r($row);
}
unset($dbh); unset($stmt);
```

3)注意:

如果PHP开启了开机自启动, 那么可能会出现连接odbc报错说库文件找不到.
这时候, 可以重启PHP那么就可以解决了, 原因暂时未明.

### php常用ODBC函数集

1. ODBC连接类函数
    odbc_connect函数: 打开一个ODBC连接
    odbc_close函数: 关闭一个已经打开的ODBC连接
    odbc_close_all函数: 关闭所有已经打开的ODBC连接
    odbc_pconnect函数: 打开一个持续有效的ODBC连接
2. ODBC操作类函数
    odbc_commit函数: 更新所有处于未决状态的操作
    odbc_do函数: 在打开的ODBC连接上执行SQL语句
    odbc_exec函数: 执行SQL语句
    odbc_execute函数: 执行一个预置的SQL语句
    odbc_free_result函数: 释放传回资料所占用的内存
    odbc_prepare函数: 预置SQL语句的执行
    odbc_rollback函数: 撤销所有处于未决状态的操作
3. ODBC信息获取类函数
    odbc_columnprivileges函数: 列出给定表的列和相关的权限
    odbc_columns函数: 列出指定表的列的名称
    odbc_cursor函数: 获取光标的名称
    odbc_data_source函数: 返回连接数据库的信息
    odbc_error函数: 获取最后的错误代码
    odbc_errormsg函数: 获取最后的错误信息
    odbc_fetch_array函数: 获取结果集数组
    odbc_fetch_into函数: 获取传回的指定列
    odbc_fetch_object函数: 返回结果集到对象
    odbc_fetch_row函数: 获取传回的一列
    odbc_field_len函数: 获取字段的长度
    odbc_field_name函数: 获取字段的名称
    odbc_field_num函数: 获取字段的序号
    odbc_field_precision函数: 获取字段的长度
    odbc_field_scale函数: 获取字段的浮点数
    odbc_field_type函数: 获取字段的资料类型
    odbc_foreignkeys函数: 返回特定表的外来键
    odbc_gettypeinfo函数: 返回数据库的类型信息
    odbc_longreadlen函数: 设定传回栏的最大值
    odbc_num_fields函数: 获取字段数目
    odbc_num_rows函数: 获取传回的列数目
    odbc_primarykeys函数: 返回列的名字作为表的主键
    odbc_procedurecolumns函数: 返回检索过程的参数信息
    odbc_procedures函数: 获取存在于特定数据源中的进程信息
    odbc_result_all函数: 传回HTML表格信息
    odbc_result函数: 获取结果数据
    odbc_specialcolumns函数: 返回一个表中在传送更新时可以自动更新的列
    odbc_statistics函数: 获取表的状态及其索引
    odbc_tableprivileges函数: 列出表格和每个表格关联的权限
    odbc_tables函数: 获取特定数据库上的表的名称
    odbc_autocommit函数: 开启或关闭自动更新
    odbc_binmode函数: 设定二进制的数据处理方式
    odbc_next_result函数: 检查下一个结果集是否可用
    odbc_setoption函数: 调整ODBC设定
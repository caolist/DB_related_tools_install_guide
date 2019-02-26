# 前言
最近工作中在调研各种数据库数据的抽取到大数据平台的方案，其中 Oracle 是一个很重要的部分，但实验平台上没有现成的 Oracle 数据库，因此需要安装，以下为安装过程的记录。
# 准备工作

1. CentOS 操作系统自行安装（64位），网络自行配置
2. 下载Oracle安装包：
linux.x64_11gR2_database_1of2.zip
linux.x64_11gR2_database_2of2.zip
下载地址：[https://www.oracle.com/technetwork/database/enterprise-edition/downloads/112010-linx8664soft-100572.html](https://www.oracle.com/technetwork/database/enterprise-edition/downloads/112010-linx8664soft-100572.html)

# 安装过程

## 配置 yum 源

```shell
$ cd /etc
$ mv yum.repos.d yum.repos.d.bak
$ mkdir yum.repos.d
$ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
$ yum clean all
$ yum makecache
```
## 安装依赖包

```shell
$ yum -y install binutils \
$ compat-libstdc++-33 \
$ elfutils-libelf \
$ elfutils-libelf-devel \
$ expat \
$ gcc \
$ gcc-c++ \
$ glibc \
$ glibc-common \
$ glibc-devel \
$ glibc-headers \
$ libaio \
$ libaio-devel \
$ libgcc \
$ libstdc++ \
$ libstdc++-devel \
$ make \
$ pdksh \
$ sysstat \
$ unixODBC \
$ unixODBC-devel
```
## 检查依赖是否安装完整
```shell
$ rpm -q \
$ binutils \
$ compat-libstdc++-33 \
$ elfutils-libelf \
$ elfutils-libelf-devel \
$ expat \
$ gcc \
$ gcc-c++ \
$ glibc \
$ glibc-common \
$ glibc-devel \
$ glibc-headers \
$ libaio \
$ libaio-devel \
$ libgcc \
$ libstdc++ \
$ libstdc++-devel \
$ make \
$ pdksh \
$ sysstat \
$ unixODBC \
$ unixODBC-devel | grep "not installed"
```
发现 pdksh 没有安装：
通过 yum install pdksh -y 安装缺少 package
通过 wget 命令直接下载 pdksh 的 rpm 包到至 /tmp/
```shell
$ wget -O /tmp/pdksh-5.2.14-37.el5_8.1.x86_64.rpm http://vault.centos.org/5.11/os/x86_64/CentOS/pdksh-5.2.14-37.el5_8.1.x86_64.rpm
```
安装 pdksh：
```shell
$ rpm -ivh pdksh-5.2.14-37.el5_8.1.x86_64.rpm
```
执行之前的命令再次检查依赖包是否安装完整

## 添加 oracle 用户组和用户
```shell
$ groupadd oinstall
$ groupadd dba
$ groupadd asmadmin
$ groupadd asmdba
$ useradd -g oinstall -G dba,asmdba oracle -d /home/oracle
```

## 查看 oracle 用户
```shell
$ id oracle
```

## 初始化 oracle 用户的密码
```shell
$ passwd oracle
```
## 配置 hostname
```shell
$ vim /etc/hosts
```
添加内容：
```shell
x.x.x.x（本机 IP） oracle
```
测试 hostname：
```shell
$ ping -c 3 oracle
```

## 优化 OS 内核参数
kernel.shmmax 参数设置为物理内存的一半
```shell
$ vim /etc/sysctl.conf
```
设置内容为：
```shell
fs.aio-max-nr=1048576
fs.file-max=6815744
kernel.shmall=2097152
kernel.shmmni=4096
kernel.shmmax = 536870912
kernel.sem=250 32000 100 128
net.ipv4.ip_local_port_range=9000 65500
net.core.rmem_default=262144
net.core.rmem_max=4194304
net.core.wmem_default=262144
net.core.wmem_max=1048586
```
使参数生效：
```shell
$ sysctl -p
```
## 限制 oracle 用户的 shell 权限
```shell
$ vim /etc/security/limits.conf
```
在末尾添加：
```shell
oracle soft nproc 2047
oracle hard nproc 16384
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft stack 10240
oracle hard stack 10240
```

```shell
$ vim /etc/pam.d/login
```

```shell
session required /lib64/security/pam_limits.so
session required pam_limits.so
```

```shell
$ vim /etc/profile
```
```shell
if [ $USER = "oracle" ]; then
    if [ $SHELL = "/bin/ksh" ]; then
        ulimit -p 16384
        ulimit -n 65536
    else
        ulimit -u 16384 -n 65536
    fi
fi
```
使之生效：
```shell
$ source /etc/profile
```
## 创建 oracle 安装目录
```shell
$ mkdir -p /db/app/oracle/product/11.2.0
$ mkdir /db/app/oracle/oradata
$ mkdir /db/app/oracle/inventory
$ mkdir /db/app/oracle/fast_recovery_area
$ chown -R oracle:oinstall /db/app/oracle
$ chmod -R 775 /db/app/oracle
```

## 配置 oracle 用户环境变量
切换 oracle 用户登陆后配置环境变量：
```shell
$ su - oracle 
$ vim .bash_profile
```
```shell
export ORACLE_HOSTNAME=oracle
export ORACLE_BASE=/db/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/
export ORACLE_SID=ORCL
export PATH=.:$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$ORACLE_HOME/jdk/bin:$PATH
export LC_ALL="en_US"
export LANG="en_US"
export NLS_LANG="AMERICAN_AMERICA.ZHS16GBK"
export NLS_DATE_FORMAT="YYYY-MM-DD HH24:MI:SS"
```
 
以上配置完成后，建议重启系统：
```shell
$ reboot
```

## 解压 oracle 压缩文件到 /db
```shell
$ cd /db/
$ unzip linux.x64_11gR2_database_1of2.zip -d /db
$ unzip linux.x64_11gR2_database_2of2.zip -d /db
```

解压完成后：
```shell
$ mkdir /db/etc/
$ cp /db/database/response/* /db/etc/
$ vim /db/etc/db_install.rsp
```

```shell
oracle.install.option=INSTALL_DB_SWONLY
DECLINE_SECURITY_UPDATES=true
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/db/app/oracle/inventory
SELECTED_LANGUAGES=en,zh_CN
ORACLE_HOSTNAME=oracle
ORACLE_HOME=/db/app/oracle/product/11.2.0
ORACLE_BASE=/db/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.isCustomInstall=true
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=dba
```
开始安装：

```shell
$ su - oracle
$ ./runInstaller -silent -ignorePrereq -responseFile /home/oracle/etc/db_install.rsp
```
安装期间可以使用tail命令监看 oracle 的安装日志：
```shell
$ tail -f /db/app/oracle/inventory/logs/installActions*.log
```
安装完成，提示 Successfully Setup Software.

使用 root 用户执行脚本：
```shell
$ su - root
$ /db/app/oraInventory/orainstRoot.sh
$ /db/app/oracle/product/11.2.0/root.sh
```
增加或修改 oracle 的环境变量：
```shell
$ su  - oracle
$ vim ~/.bash_profile
```
```shell
export ORACLE_BASE=/db/app/oracle
export ORACLE_SID=orcl
export ROACLE_PID=ora11g
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/usr/lib
export ORACLE_HOME=/db/app/oracle/product/11.2.0/
export PATH=$PATH:$ORACLE_HOME/bin
export LANG="zh_CN.UTF-8"
export NLS_LANG="SIMPLIFIED CHINESE_CHINA.AL32UTF8"
export NLS_DATE_FORMAT='yyyy-mm-dd hh24:mi:ss'
```
## 配置监听程序
```shell
$ netca /silent/responsefile/home/oracle/etc/netca.rsp
```
## 查看监听端口
```shell
$ netstat -tnulp | grep 1521
```
## 启动监控程序
```shell
$ lsnrctl start
```
## 静默创建数据库

```shell
$ vim /etc/dbca.rsp
```
TOTALMEMORY 设置为总内存的 80%

```shell
GDBNAME = "orcl"
SID = "orcl"
SYSPASSWORD = "oracle"
SYSTEMPASSWORD = "oracle"
SYSMANPASSWORD = "oracle"
DBSNMPPASSWORD = "oracle"
DATAFILEDESTINATION =/db/app/oracle/oradata
RECOVERYAREADESTINATION=/db/app/oracle/fast_recovery_area
CHARACTERSET = "AL32UTF8"
TOTALMEMORY = "1638"
```

## 执行静默建库
```shell
$ dbca -silent -responseFile /db/etc/dbca.rsp
```

## 查看 oracle 实例进程
```shell
$ ps -ef | grep ora_ | grep -v grep
```

## 查看监听状态
```shell
$ lsnrctl status
```

## 登录 sqlplus

查看实例状态：
```shell
$ sqlplus / as sysdba
select status from v$instance;
```
查看数据库编码：
```shell
select userenv('language') from dual;
```
查看数据库版本信息：
```shell
select * from v$version;
```
激活 scott 用户：
```shell
alter user scott account unlock;
alter user scott identified by tiger;
select username,account_status from all_users;
```

## 防火墙开放1521端口
```shell
$ firewall-cmd --zone=public --add-port=1521/tcp --permanent
$ firewall-cmd --reload
```

# 总结
在 Linux 上安装配置 Oracle 确实是一个比较繁复的过程，耗时比较多，但纵观上下其实流程还是很清晰的，只要明白其中原理，细心执行步骤，安装成功水到渠成。
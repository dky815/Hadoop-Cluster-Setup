# hadoop-2.7.1+hbase-1.1.5+zookeeper-3.4.5+hive-1.2.1完全分布式搭建

# **一．方案计划说明**
## **1.1 环境说明：**
搭建hadoop集群环境至少需要3个节点（也就是3台服务器设备）：1个Master，2个Slave。
节点之间局域网连接，可以相互ping通，各节点IP如下：

|    Hostname      |    IP      |    User      |    Password      | 
|:----:|:----:|:----:|:----:|
|    16211109-master      |    10.251.254.87      |    root      |    098765      | 
|    16211112-slave1      |    10.251.254.252      |    root      |    098765      | 
|    16211130-slave2      |    10.251.254.250      |    root      |    098765      | 

三个节点均使用CentOS 7系统，为了便于维护，集群环境配置项一切涉及密码登录的地方均使用相同用户名、相同用户密码、相同hadoop、zookeeper、hbase、hive目录结构。
## 1.2 **版本兼容性方案：**
我们采用的各组件版本为：

JDK ：jdk1.8.0_201

Hadoop ：hadoop-2.7.1

Hbase ：hbase-1.1.5-bin

Zookeeper ：zookeeper-3.4.5

Hive ：hive-1.2.1-bin
## **1.3 计划的各节点角色及服务进程：**

|    Hostname    |    IP    |    节点角色及进程    | 
|:----:|:----:|:----|
|    16211109-master    |    10.251.254.87    | **Hadoop:** master(NameNode、SecondaryNameNode、ResourceManager) **ZooKeeper:** follower(QuorumPeerMain) **Hbase:** master(HMaster、HRegionServer)   |    | 
|    16211112-slave1    |    10.251.254.252    | **Hadoop:** slave(DataNode、NodeManager)   **ZooKeeper:** follower(QuorumPeerMain) **Hbase:** slave(HRegionServer) |    | 
|    16211130-slave2    |    10.251.254.250    | **Hadoop:** slave(DataNode、NodeManager)   **ZooKeeper:** leader(QuorumPeerMain) **Hbase:** slave(HRegionServer) |    | 

## **1.4 hadoop集群各组件部署配置顺序：**
因为hadoop集群的组件启动顺序与配置顺序都有着严格的要求：

启动顺序：hadoop->zookeeper->hbase>hive

关闭顺序：hive->hbase->zookeeper->hadoop

为了最大程度避免配置过程出错，所以我们配置各组件顺序也是按照启动顺序进行。
## **1.5 其他需要考虑的事项**
通过本学期的大数据课程我们了解到，计算机集群的基本架构如图：

![图片](https://uploader.shimo.im/f/lG6puDBw0fILiQ27.png!thumbnail)

一个机架中运行着多个节点，一个集群由多个机架组成。

同一机架中的节点通过网络互联，不同机架的节点通过交换机互联。

所以我们在选定机器搭建hadoop集群环境时也要将计算机集群的特点利用起来，所有的节点最好两两分布于不同的机架上。这样的话，如果某机架发生意外情况导致机架内所有节点不可逆损坏，受到影响的节点数量将是一个最低的值，从而可以最大程度得保证更多节点的运行，使集群系统尽最大可能正常提供服务，提高系统的可用性和容错性。

若较多的节点位于同一机架，则该机架若损坏将可能导致过多节点宕机进而整个集群系统彻底不可用，应该极力避免这种情况。

如果有条件的话，甚至可以不局限于物理空间限制，将集群的各个节点分散部署在全国乃至全球各地的机房，这样可以极大提高可用性和可靠性。

# **二．准备工作**
## **2.1 添加hosts映射关系**
分别在三个节点上添加hosts映射关系:：

$ vim /etc/hosts

添加内容如下：

10.251.254.87 16211109-master

10.251.254.252 16211112-slave1

10.251.254.250 16211130-slave2

## **2.2 集群之间ssh无密码登录**
CentOS默认安装了ssh，如果没有需要先安装ssh。

集群环境的使用必须通过ssh无密码登陆来执行，本机登陆本机必须无密码登陆，主机与从机之间必须可以双向无密码登陆，从机与从机之间无限制。
### **2.2.1 开启Authentication免登陆**
CentOS默认没有启动ssh无密登录,编辑$ vim /etc/ssh/sshd_config，去掉以下两行注释，开启Authentication免登陆。

\# RSAAuthentication yes

\# PubkeyAuthentication yes

如果是root用户下进行操作，还要去掉 \# PermitRootLogin yes的注释，允许root用户登录。
### **2.2.2 生成authorized_keys**
输入命令，$ ssh-keygen -t rsa，生成key，一直按回车。

就会在/root/.ssh生成：authorized_keys   id_rsa.pub   id_rsa 三个文件，为了各个机器之间的免登陆，在每一台机器上都要进行此操作。
### **2.2.3 合并公钥到authorized_keys文件**
master节点进入/root/.ssh目录，输入以下命令：

$ cat id_rsa.pub>> authorized_keys

#### 把master公钥合并到authorized_keys中

$ ssh root@10.251.254.252 cat ~/.ssh/id_rsa.pub>> authorized_keys

$ ssh root@10.251.254.250 cat ~/.ssh/id_rsa.pub>> authorized_keys

#### 把slave1、slave2公钥合并到authorized_keys中

完成之后输入命令，把authorized_keys远程copy到slave1和slave2之中

$ scp authorized_keys 10.251.254.252:/root/.ssh/

$ scp authorized_keys 10.251.254.250:/root/.ssh/

最好在每台机器上进行$ chmod 600 authorized_keys操作，使当前用户具有 authorized_keys的读写权限。（见7.1）

拷贝完成后，在每台机器上进行 service sshd restart  操作， 重新启动ssh服务。

在每台机器输入 ssh 10.251.254.xxx，测试能否无需输入密码连接另外两台机器。

## **2.3 安装jdk**

新建目录：$ mkdir /root/apps

解压安装：$ tar -zxf jdk.tar.gz

修改配置文件：$ vim /etc/profile，配置环境变量

export JAVA_HOME=/root/apps/jdk1.8.0_201

export PATH=$JAVA_HOME/bin:$PATH

重新加载配置文件使之生效$ source /etc/profile

## **2.4 关闭防火墙和selinux**
$ systemctl stop firewalld.service

$ systemctl disable firewalld.service

$ vim /etc/sysconfig/selinux

SELINUX=enforcing 改为 SELINUX=disabled

$ reboot
# **三.  hadoop集群安装配置**
## **3.1 安装hadoop（只在master节点操作，配置完成后scp分发至slave节点）**
进入apps目录：& cd /root/apps

解压安装：$ tar -zxf hadoop-2.7.1.tar.gz

$ mv hadoop-2.7.1 hadoop

## **3.2 修改环境变量（三个节点均需要操作）**
修改配置文件：$ vim /etc/profile，配置环境变量

export JAVA_HOME=/root/apps/jdk1.8.0_201

export HADOOP_HOME=/root/apps/hadoop

export HADOOP_INSTALL=$HADOOP_HOME

export HADOOP_MAPRED_HOME=$HADOOP_HOME

export HADOOP_COMMON_HOME=$HADOOP_HOME

export HADOOP_HDFS_HOME=$HADOOP_HOME

export YARN_HOME=$HADOOP_HOME

export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native

export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin

重新加载配置文件使之生效$ source /etc/profile

## **3.3 修改hadoop配置文件（只在master节点操作，配置完成后scp分发至slave节点）**
配置hadoop的配置文件：core-site.xml&emsp;hdfs-site.xml&emsp;mapred-site.xml&emsp;yarn-site.xml&emsp;slaves

$cd /root/apps/hadoop/etc/hadoop

$vim core-site.xml

**1.core-site.xml**
```
<configuration>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://16211109-master:9000</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>file:/root/apps/hadoop/tmp</value>
  </property>
</configuration>
```
**2.hdfs-site.xml**
```
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/root/apps/hadoop/tmp/dfs/name</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/root/apps/hadoop/tmp/dfs/data</value>
  </property>
  <property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>16211109-master:9001</value>
  </property>
</configuration>
```
**3.mapred-site.xml**
```
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```
**4.yarn-site.xml**
```
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>16211109-master</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>604800</value>
  </property>
</configuration>
```

**5.slaves**

16211112-slave1

16211130-slave2
## **3.4 scp分发至各个slave节点**
$scp -r /root/apps/hadoop root@10.251.254.252:/root/apps/

$scp -r /root/apps/hadoop root@10.251.254.250:/root/apps/
## **3.5 hadoop启动**
master节点：

格式化namenode：$ hadoop namenode -format

切换至脚本目录：cd /root/apps/hadoop/etc/hadoop

启动hadoop：$ ./start-all.sh（见7.2）
## **3.6 各节点查看进程状态**
启动成功各节点通过$ jps查看进程：

![图片](https://uploader.shimo.im/f/KbdC55OZuYENvEYD.png!thumbnail)

![图片](https://uploader.shimo.im/f/9LlImEwOwjIB7yS0.png!thumbnail)

![图片](https://uploader.shimo.im/f/Zcr9NXsOmHIqvRJr.png!thumbnail)
## **3.6 master节点查看管理页面**
 ![图片](https://uploader.shimo.im/f/Yi6A8sSDExgcT00j.png!thumbnail)
# **四.  Zookeeper集群安装配置**
## **4.1 安装Zookeeper（只在master节点操作，配置完成后scp分发至slave节点）**
进入apps目录：& cd /root/apps

解压安装：$ tar -zxf zookeeper-3.4.5.tar.gz

$ mv zookeeper-3.4.5 zookeeper
## **4.2 修**改配置文件zoo.cfg
进入/root/apps/zookeeper/conf目录

$ cd /root/apps/zookeeper/conf

拷贝zoo_sample.cfg文件为zoo.cfg

$ cp zoo_sample.cfg zoo.cfg

修改：$ vim zoo.cfg

server.1=10.251.254.87:2888:3888

server.2=10.251.254.252:2888:3888

server.3=10.251.254.250:2888:3888
## **4.3 新建并编辑myid文件**
新建myid文件

$ vim /root/apps/zookeeper/data/myid

输入一个数字（master为1，slave1为2，slave2为3）
## **4.4 scp分发至各个slave节点**
$scp -r /root/apps/zookeeper root@10.251.254.252:/root/apps/

$scp -r /root/apps/zookeeper root@10.251.254.250:/root/apps/

修改每个节点上myid文件中的数字（master为1，slave1为2，slave2为3）

$ vim /root/apps/zookeeper/data/myid
## **4.5 启动zookeeper集群**
对所有节点：

$ cd /root/apps/zookeeper/bin

$ ./zkServer.sh start
## **4.6 成功标识**
查看状态：

$ ./zkServer.sh status

$ jps

![图片](https://uploader.shimo.im/f/7ueg9GSo1AYDaeGN.png!thumbnail)

![图片](https://uploader.shimo.im/f/Imw1Mvkd4HsDahLM.png!thumbnail)

![图片](https://uploader.shimo.im/f/FEik2er4ScYD4vUZ.png!thumbnail)
# **五.  hbase集群安装配置**
## **5.1 安装hbase（只在master节点操作，配置完成后scp分发至slave节点）**
进入apps目录：& cd /root/apps

解压安装：$ tar -zxf hbase-1.1.5-bin.tar.gz

$ mv hbase-1.1.5-bin hbase

## **5.2 修改配置文件**
配置hbase的配置文件&emsp;hbase-env.sh&emsp;hbase-site.xml&emsp;regionservers

$cd /root/apps/hbase/conf

$vim hbase-env.sh

**1.hbase-env.sh**

export JAVA_HOME=/root/apps/jdk1.8.0_201

export HBASE_CLASSPATH=/root/apps/hadoop/etc/hadoop/

export HBASE_MANAGES_ZK=false

**2.hbase-site.xml**
```
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://10.251.254.87:9000/hbase</value>
  </property>
    <property>
        <name>hbase.master</name>
        <value>16211109-master</value>
    </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
    <property>
        <name>hbase.zookeeper.property.clientPort</name>
        <value>2181</value>
    </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>16211109-master,16211112-slave1,16211130-slave2</value>
  </property>
    <property>
        <name>zookeeper.session.timeout</name>
        <value>60000000</value>
    </property>
    <property>
        <name>dfs.support.append</name>
        <value>true</value>
    </property>
</configuration>
```

**3.regionservers**

16211109-master

16211112-slave1

16211130-slave2
## **5.3 scp分发至各个slave节点**

$scp -r /root/apps/hbase root@10.251.254.252:/root/apps/

$scp -r /root/apps/hbase root@10.251.254.250:/root/apps/
## **5.4 master节点启动hbase**
$ /root/apps/hbase/bin/start-base.sh（见7.3）
## **5.5 jps查看各节点进程状态**
![图片](https://uploader.shimo.im/f/iS6ptcHDolUzgPEi.png!thumbnail)

![图片](https://uploader.shimo.im/f/5RDRedy2eY8HKwgQ.png!thumbnail)

![图片](https://uploader.shimo.im/f/14bQTMUnbv8pyeSZ.png!thumbnail)
# **六.  hive集群安装配置**
## **6.1 安装hive**
进入apps目录：& cd /root/apps

解压安装：$ tar -xvf apache-hive-1.2.1-bin.tar.gz

$ mv apache-hive-1.2.1-bin hive

## **6.2 修改环境变量**
编辑配置文件：$ vim /etc/profile，修改环境变量

export HIVE_HOME=/root/apps/hive

export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$HIVE_HOME


重新加载配置文件使之生效$ source /etc/profile

## **6.3 修改配置文件**
进入/root/apps/hive/conf目录

$ cd /root/apps/hive/conf

拷贝hive-env.sh.template文件为hive-env.sh

$ cp hive-env.sh.template hive-env.sh

编辑修改：$ vim hive-env.sh

HADOOP_HOME=/root/apps/hadoop
## **6.4 启动hive并测试**
$ hive

启动后新建一个表测试：

![图片](https://uploader.shimo.im/f/DwlDmzkVi5sYlYRS.png!thumbnail)

## **6.5 安装mysql并配置**

解压：$ tar -xvf mysql-5.7.22-1.el6.x86_64.rpm-bundle.tar

依次执行：

$ sudo rpm -ivh --force mysql-community-common-5.7.16-1.el6.x86_64.rpm 

$ yum remove mysql-libs

$ sudo rpm -ivh --force mysql-community-libs-5.7.16-1.el6.x86_64.rpm（见7.4）

$ sudo rpm -ivh --force mysql-community-client-5.7.16-1.el6.x86_64.rpm

$ sudo rpm -ivh --force --nodeps mysql-community-server-5.7.16-1.el6.x86_64.rpm（见7.5）

## **6.6 启动mysql**
$ service mysqld start

在log文件中查找root密码：$ vim /var/log/mysqld.log

2019-07-01T10:21:48.360592Z 1 [Note] A temporary password is generated for root@localhost: .05em(zeKw1j


登录mysql：$ mysql -u root -p

输入密码：.05em(zeKw1j

修改密码：（见7.6）

mysql> set global validate_password_policy=0;

mysql> set global validate_password_length=4;

mysql> set password=password('000000');

创建hive用户：

mysql> CREATE USER 'hive'@'%' IDENTIFIED BY '000000';

mysql> GRANT ALL PRIVILEGES ON *.* TO 'hive'@'%' IDENTIFIED BY '000000';

mysql> FLUSH PRIVILEGES;

## **6.7 配置hive**

解压mysql-connector-java：

$ tar -zxf mysql-connector-java-5.1.40.tar.gz

复制jar包到hive的lib目录下：

$ cp mysql-connector-java-5.1.40-bin.jar /root/apps/hive/lib

进入/root/apps/hive/conf目录

$ cd /root/apps/hive/conf

新建：$ vim hive-site.xml

添加如下内容：
```
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true</value>
        <description>JDBC connect string for a JDBC metastore</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
        <description>Driver class name for a JDBC metastore</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hive</value>
        <description>username to use against metastore database</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>000000</value>
        <description>password to use against metastore database</description>
    </property>
</configuration>
```

## **6.8 运行hive实例并测试**

$ hive

![图片](https://uploader.shimo.im/f/CN286k5mSE8CB0cB.png!thumbnail)
# **七.  遇到的问题及解决方案**
## **7.1 scp命令发送时显示“没有那个目录”**
解决方案：

对于各个节点：

.ssh目录需要700权限

$ sudo chmod 700 ~/.ssh

.ssh目录下的authorized_keys文件需要600或644权限

$ sudo chmod 644 ~/.ssh/authorized_keys
## **7.2 执行/root/apps/hadoop/etc/hadoop/start-all.sh启动hadoop时报错：“Error:JAVA_HOME is not set and could not be found”**

解决方案：

对于各个节点：

$ vim /root/hadoop/etc/hadoop/hadoop-env.sh

将&emsp;export JAVA_HOME=$JAVA_HOME

修改为&emsp;export JAVA_HOME=/root/apps/jdk1.8.0_201

然后再次执行脚本start-all.sh即可

## **7.3 执行/root/apps/hbase/bin/start-base.sh后某节点无HRegionServer**
![图片](https://uploader.shimo.im/f/Jjaf1SbjcSc9hlo9.png!thumbnail)

![图片](https://uploader.shimo.im/f/thfIjFjs0zotldfy.png!thumbnail)

由于节点间时间不同步所致：

![图片](https://uploader.shimo.im/f/WRH7oeLwF145B4v5.png!thumbnail)

![图片](https://uploader.shimo.im/f/Vrn7mzX5t7wSss24.png!thumbnail)

![图片](https://uploader.shimo.im/f/ujiB2oTJTsgzaJQn.png!thumbnail)

解决方案：各节点安装ntp服务使时间与网络同步

$ yum -y install ntp ntpdate

$ ntpdate cn.pool.ntp.org

## **7.4 ****安装mysql****-libs****时****报****错****：错误：依赖检测失败：mariadb-libs 被 mysql-community-libs-5.7.22-1.el6.x86_64 取代**

解决方案：

使用$ yum remove mysql-libs命令删除原本的mysql-libs

重新安装即可

## **7.5 安装mysql-server时报错：错误：依赖检测失败：libsasl2.so.2()(64bit) 被 mysql-community-server-5.7.22-1.el6.x86_64 需要**
解决方案：

安装命令加两个参数即可：

$ sudo rpm -ivh mysql-community-server-5.7.22-1.el6.x86_64.rpm --force --nodeps
## **7.6 修改mysql密码时报错：ERROR 1819 (HY000): Your password does not satisfy the current policy requirements**
解决方案：

先修改密码安全等级：

mysql> set global validate_password_policy=0;

mysql> set global validate_password_length=4;

再修改密码：

mysql> set password=password('000000');



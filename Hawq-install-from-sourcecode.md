# Hawq源码安装遇到的问题
@(hawq源码安装)[遇到的问题|邮箱 qingxinwhu@qq.com]

---

源码安装Hawq，官方给的连接: [buidl&&install](https://cwiki.apache.org/confluence/display/HAWQ/Build+and+Install),我的环境是Centos6，但是在安装过程中出现一些问题，Centos7出现问题参考如下步骤应该也能解决。我的环境：Centos6,三个节点，一个Mster节点两个Segment节点。Master节点和Hadoop的namespace节点是同一个节点，两个Segment节点分别是Hadoop(HDFS)的datanode的节点。


1. 首先安装Hadoop环境，需要安装jave环境，参考这位童鞋的博客:[Hadoop集群安装配置教程_Hadoop2.6.0_Ubuntu/CentOS](http://www.powerxing.com/install-hadoop-cluster/)

2. 安装依赖库：有些通过源码安装，有些通过yum安装。安装epel-release后需要更改下yum的配置文件。还需要注意，通过源码安装的库(包括下面编译的depends/libyarn/的库)如果在./configure的时候不加--prefix选项被默认安装到/usr/local/lib下面，有的系统在编译链接找共享库的时候不去这个目录搜索，则编译hawq源码会出现例如:不能加载共享库libhdfs3.so.1之类的报错，解决方法是要么将build好的库拷贝到/usr/lib下要么添加系统搜索目录项，在/etc/ld.so.conf.d/目录下新建hawq.conf文件，内容是/usr/local/lib，在/ld.so.conf.d/目录下执行
- #vim hawq.conf    //添加内容
- #ldconfig         //使配置文件生效
3. 除了官方给的库以外经过安装实践还需要安装以下库：根据Docker下安装步骤https://hub.docker.com/r/mayjojo/hawq-devel/~/dockerfile/
   - #yum install python-pip
   - #yum install maven
   - 安装python模块# yum makecache && yum install -y postgresql-devel && \
pip --retries=50 --timeout=300 install pg8000 simplejson unittest2 pycrypto pygresql pyyaml lockfile paramiko psi && \
pip --retries=50 --timeout=300 install http://darcs.idyll.org/~t/projects/figleaf-0.6.1.tar.gz && \
pip --retries=50 --timeout=300 install http://sourceforge.net/projects/pychecker/files/pychecker/0.8.19/pychecker-0.8.19.tar.gz/download && \
yum erase -y postgresql postgresql-libs postgresql-devel && \
yum clean all
4. 配置系统参数 OS requirement 按照官方教程，没问题。
5. 从官方教程中给的github地址下载hawq源码，按照教程先编译libyarn，没有问题。再编译hawq源码:
- # ./configure --prefix=/hawq  安装到/hawq目录下 如果没有配置共享库搜索目录/usr/local/lib会出现libhdfs找不到的问题，还会出现test sample过不去的问题。
- # make    会下载很多jar包，网络状况不佳会超时出错。
- # make install
6. 以上操作需要在所有节点上执行，在每个节点上安装hawq。
7. 创建用户gpadmin用来登录hawq
- #useradd -s /bin/bash gpadmin
- #passwd gpadmin
- #chown -R hawqadmin.hawqadmin /opt/hawq/   
改变hawq所有者权限用gpadmin用户运行
8. 配置gpadmin用户的sshkey用来Master无密码登录两个segment节点
也就是在Master节点上ssh segment1， ssh segment2不需要密码直接登录。
9. 在安装hadoop阶段应该已经在hosts文件设置了IP和别名 例如 
XXX.XXX.XXX.XXX Master
XXX.XXX.XXX.XXX Segment1
XXX.XXX.XXX.XXX Segment2
10. 在配置hawq前确认hdfs是可以正常运行的，参见[这里](http://hawq.docs.pivotal.io/130/docs-hawq/topics/InstallingHAWQ.html#preparingtoinstallhawq)http://hawq.docs.pivotal.io/130/docs-hawq/topics/InstallingHAWQ.html#preparingtoinstallhawq 中Prepare HDFS章节的第8步骤To verify that HDFS has started。
注意：例如hadoop fs -ls / 命令是ls出hdfs中根目录 hdfs维护了自己的空间
 # hadoop fs -mkdir /test  在hdfs空间中创建/test目录
11. 在启动前改写hawq的配置文件/hawq/etc/hawq-site.xml 和  /hawq/etc/hawq-client.xml
- hawq-site.xml文件关键的配置项
```python
  <property>
        <name>hawq_master_address_host</name>
        <value>Master</value> 
        <description>The host name of hawq master.</description>
  </property>
  <property>
        <name>hawq_standby_address_host</name>
        <value>none</value>
        <description>The host name of hawq standby master.</description>
  </property>
  <property>
        <name>hawq_dfs_url</name>
        <value>Master:9000/hawq_default</value>
        <description>URL for accessing HDFS.</description>
  </property>
  <property>
        <name>hawq_master_directory</name>
        <value>/data/master</value>
        <description>The directory of hawq master.</description>
  </property>
  <property>
        <name>hawq_segment_directory</name>
        <value>/data/primary</value>
        <description>The directory of hawq segment.</description>
  </property>
  
  ```
   hawq_master_address_host：代表主节点，Master是hosts文件中的别名
   hawq_standby_address_host：表示主节点的备份节点，我们没有则为none
   hawq_dfs_url：表示hawq的hdfs目录，我们设置为/hawq_default,下面会创建
   hawq_master_directory：表示hawq(前身是postgresql)在Master的元数据位置，其实也是postgresql的initdb的位置
   hawq_segment_directory：表示hawq的segment节点的元数据位置
  - hawq-client.xml
  ```python
 <property>
        <name>dfs.nameservices</name>
        <value>hawqcluster</value>
 </property>
<property>
        <name>dfs.ha.namenodes.hawqcluster</name>
        <value>nn1</value>
</property>
<property>
        <name>dfs.namenode.rpc-address.hawqcluster.nn1</name>
        <value>Master:9000</value>
</property>
<property>
        <name>dfs.namenode.http-address.hawqcluster.nn1</name>
        <value>Master:51200</value>
</property> 
 
  ```
  **注意**：将配置文件hawq-site.xml hawq-client.xml scp到其他segment节点上，或者去每个segment上改同样的配置，保证所有节点的配置是一样的。
  
  下面创建配置文件中列出的目录：
  - #hdfs dfs -mkdir /hawq_default && hdfs dfs -chown gpadmin /hawq_default
  创建/hawq_default目录
  - 在Master节点上创建/data/master在其他sement节点上创建/data/primary目录
  记得将目录更改为gpadmin用户所有。
12. 用gpadmin用户登录Master，开始初始化hawq
 - #cd /hawq
 - #source greenplum_path.sh
 - #hawq init cluster
 没有第3步骤，init时会出现python pgdb模块没找到的报错。
13. 切换到gpadmin用户就可以登录了#psql -d postgres
14. 如果没有成功请我碰到的这几个小问题：
- 配置文件同步到其他节点上没有，所有hawq节点保持同样的配置文件内容
- pgadmin用户是否能从Master节点无密码登录到其他segment节点上
- 其他segment节点是否正确能正确找到需要的库，例如，如果segment节点没有初始化成功去/home/gpadmin/hawqAdminLogs/目录下查看对应日志报错找原因，例如在segment节点下/hawq/bin中#./postgres -V 能输出版本信息则hawq安装没问题。
- 是不是没有运行ldconfig (我就忘了这一步，导致segment初始化没成功)
- 如果没有初始化成功，需要第二次重新执行，记得一下几点：
  1. 再次初始化前清空 /data/master /data/primary /hawq_default 下的内容
  清空hawq_default:#hdfs dfs -rm -r /hawq_default/*
  2. 再次初始化前确定所有节点上没有postgres进程在运行，通过#ps -e |grep post 查看，如果有kill -9掉postgres进程
  3. 完成1，2步骤才能再次init
  
15. 多用google搜索，参考一下几个网址：
- https://cwiki.apache.org/confluence/display/HAWQ/Build+and+Install
- https://hub.docker.com/r/mayjojo/hawq-devel/~/dockerfile/
- http://hawq.docs.pivotal.io/130/docs-hawq/topics/InstallingHAWQ.html
这是在其他平台上rpm、二进制安装不是源码，但有一点信息是值得参考的。主要参考前两个连接，去连接的其他tab上看看也许能发现问题。
16. GOOD LUCK!!


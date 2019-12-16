### 第一章 前言和部署说明

##### 1.1Hadoop大数据说明

~~~markdown
大数据是一个生态体系概念，而Hadoop只是大数据生态体系中的一个技术架构。Hadoop采用HDFS文件系统（Hadoop Distributed File System，Hadoop分布式文件系统）进行海量数据的存储，然后通过不同的应用或者技术进行数据分析和结果展现。
~~~

​         Hadoop以Job定义不同的数据分析和处理需求，再用任务来拆分这些需求，然后将任务分配到各个服务器上进行分布式运算，然后再将各服务器上计算的结果汇总起来行形成最终的结果，这个就是hadoop最核心的数据计算和分析技术架构：**Mapreduce，同时也是*Hadoop1.x*版本的任务调度架构**. 正是在这样的架构下，我们可以用大量廉价的x86服务器作为hadoop的数据计算节点，通过海量分布式计算的方式来进行海量数据的处理。  

​		**到了Hadoop2.x版本**，Hadoop采用了新的任务调度架构：**Yarn**，采用更加高效的任务调度方式，使得Hadoop集群可以扩展到比1.x版本更大的数据处理节点规模，从而极大的提升了数据处理规模，而之前的**Mapreduce从任务调度架构也进化成了Hadoop数据处理应用**，仍然是Hadoop上最核心的数据处理方式。

​		Hadoop能够进行分布式数据处理依赖于HDFS分布式文件系统。HDFS是Hadoop体系中数据存储管理的基础。它是一个**高度容错**的系统，能检测和应对硬件故障，用于在低成本的通用硬件上运行。HDFS简化了文件的一致性模型，**通过流式数据访问**，**提供高吞吐量应用程序数据访问功能**，适合带有大型数据集的应用程序。



#### HDFS适合做什么/不适合做什么

| 存储海量数据：TB、PB级数据       | 存储和处理小文件：数据存储效率和任务处理效率不高 |
| -------------------------------- | ------------------------------------------------ |
| 非结构化数据                     | 大量的随机读：与存储介质有关                     |
| 注重数据处理吞吐量，对延时不敏感 | 不支持文件并发写入：HDFS采用的流式写入           |
| 适用一次写入多次读取的场景       | 需要对文件修改：不支持文件修改                   |



##### 1.2 工作中acloud（HCI）部署Hadoop说明

**Hadoop在数据处理过程中对网络带宽、磁盘吞吐、CPU和内存都有一定的要求，具体表现为：**

​         1)网络带宽：**网络资源需求主要表现在数据插入阶段和Mapreduce中的reduce-shuffle阶段**。  在HDFS多副本（2副本或以上）下，**每次往HDFS插入或者回写数据时，数据块本地写完后都需要再通过网络拷贝副本到其他节点，所以副本数越多，数据量越大，对网络带宽的消耗也是非常大的**，所以hadoop节点虚拟机间数据通信网络推荐用**万兆网络**。

​		2)磁盘吞吐：hadoop上跑数据处理的过程中（通常为Mapreduce应用），都是**顺序的读写**，并且已读居多。HDFS默认情况下块大小为64MB，数据量较大情况下会设置为128MB、256MB甚者512MB，所以这样的场景下对存储的吞吐性能要求较高，在部署hadoop数据节点时，**推荐采用10K-SAS盘做数据盘，而且不需要SSD缓存**。

​		3)CPU：在数据处理阶段多个任务并发执行指定数据处理算法的运算时，需要大量的CPU资源，因此在配置生产环境的hadoop集群的HCI主机时，默认采用aServer2205或以上型号，在配置培训、测试环境的hadoop集群的HCI主机时，默认采用aServer2105或以上型号。

​		4)内存：内存对hadoop**最直接的影响就是限制并发运行的任务数**。每个任务或者容器（**Yarn任务调度架构下每个任务分配一个容器**）都会消耗内存，这个由任务需求和最大限制参数有关，在启动任务或容器时，**hadoop会根据可用内存和每个任务使用内存计算出总共能够并行多少个任务**，在CPU、磁盘、网络资源够用的情况下，启动的任务越多，数据处理的速度也就越快。因此尽量不要让内存资源紧缺导致启动的任务数不足使得服务器资源利用不正确，通常情况下配置hadoop集群的HCI主机内存要比其他业务主机内存要大，具体配置需要根据具体需求来定。

   根据以上四点说明，我们可以推荐一个生产环境下默认的HCI主机最低配置：

| 类型说明  | 配置说明                        | 备注 |
| --------- | ------------------------------- | ---- |
| 主机型号  | aServer2205                     |      |
| 主机配置  | CPU:2*Intel Xeon 2650v4,12C,24T |      |
| 内存      | 256GB                           |      |
| HCI系统盘 | 128GB SSD（内置）               |      |
| SSD缓存盘 | 2*480GB Intel SSD               |      |
| HDD数据盘 | 10*1.8TB 10K SAS                |      |

###### 1.2.1  hci（acloud） Hadoop大数据架构说明  

###### 1.2.1.1         部署Hadoop资源池说明  

​		在一个Hadoop大数据集群中，除了用于进行大数据分析的业务外，还有大数据数据管理和结果报表展示的业务。不同的业务对资源要求也不同，我们在HCI上将资源进行了池化，将不同的业务划分到不同的资源池中，通过深信服HCI管理平台进行统一的监控和管理。

​		资源池按照存储进行划分：**Hadoop资源池和aSAN资源池**。在实际大数据生产或测试环境的部署中，也是先进行资源池的划分和容量规划，然后再进行部署。

Hadoop资源池    aSan资源池

运行业务     Hadoop数据处理相关业务的虚拟机：

1.Hadoop Namenode节点

2.Hadoop Datanode节点    

Hadoop数据管理和报表分析相关的虚拟机、其他非大数据处理应用虚拟机：

1.Hadoop集群管理节点（发行版）

2.大数据应用API网关节点

3.大数据数据处理结果报表节点

4.其他非大数据处理应用节点

存储划分（以aServer 2205作为示例）   

1.1*1.8T 10K-SAS 做Name/Datanode虚拟机系统盘

2.6*1.8T 10K-SAS做Datanode数据盘 1.1*480GB SSD做aSAN缓存

2.4*1.8T 10K-SAS做aSAN数据盘

CPU划分 按需划分，HCI需要预留，不建议超配   按需划分，HCI需要预留，不建议超配

内存划分     按需划分，HCI需要预留   按需划分，HCI需要预留



###### 1.2.1.2Hadoop控制与数据节点**整合部署架构**

这样的部署是hadoop集群规模不大，通常在10台主机以内的情况下，采用此架构部署。

###### 1.2.1.3Hadoop控制与数据节点**分开部署架构**

这样的部署在hadoop集群规模较大，通常在10台以上，20台或以下的情况下的部署。

###### 1.2.1.4Hadoop集群部署在HCI上最大规模参考标准: hci部署hadoop集群最大规模参考



##### 1.2.2sangfor-HCI部署Hadoop大数据价值说明

**略**





### 第2章 Hadoop大数据部署规划

​		本文以在HCI上部署Apache hadoop原生版本作为示例，展示如何在Hadoop上创建和部署hadoop集群，并通过Hadoop上的Mapreduce示例程序验证Hadoop集群的可用性。



​         **下面主要介绍Hadoop资源池的部署**  

##### 2.1 图   略

#####  2.2Hadoop节点与IP规划

| 节点信息          | 主机域名   | IP地址配置                                                   | 网卡名称     | 备注                                                         |
| ----------------- | ---------- | :----------------------------------------------------------- | ------------ | ------------------------------------------------------------ |
| Namenode          | hadoop.nn  | IP：192.168.1.101                                                                    Mask：255.255.255.0                                                                      GateWay：192.168.254 | eth0    eht1 | Namenode节点用于管理Hadoop集群文件系统的元数据和与客户端交互，即hadoop集群的管理节点 |
| SecondaryNamenode | hadoop.sn  | IP：192.168.1.102                         Mask：255.255.255.0                          GateWay：192.168.254 | eth0         | SecondaryNamenode节点用于从Namenode备份元数据，并且定时创建checkpoint并维护元数据更新记录文件EditLog，即hadoop集群的**辅助管理节点** |
| Datanode1         | hadoop.dn1 | IP：192.168.1.1        Mask：255.255.255.0 GateWay：192.168.254 | eth0         | Datanode节点是Hadoop集群的数据处理节点，即**真正的大数据分析工作节点** |
| Datanode2         | hadoop.dn2 | IP：192.168.1.3                Mask：255.255.255.0 GateWay：192.168.254 | eth0         |                                                              |
| Datanode3         | hadoop.dn3 | IP：192.168.1.3                 Mask：255.255.255.0 GateWay：192.168.254 | eth0         |                                                              |
| Datanode4         | hadoop.dn4 | IP：192.168.1.4             Mask：255.255.255.0 GateWay：192.168.254 | eth0         |                                                              |
| Datanode5         | hadoop.dn5 | IP：192.168.1.5                Mask：255.255.255.0  GateWay：192.168.254 | eth0         |                                                              |
| Datanode6         | hadoop.dn6 | IP：192.168.1.6               Mask：255.255.255.0 GateWay：192.168.254 | eth0         |                                                              |



##### 2.3Hadoop集群配置规划

###### 2.3.1硬件配置规划

在HCI上部署Hadoop推荐采用12盘位的服务器，

###### 2.3.2虚拟机配置规划

###### 2.3.3Hadoop集群配置规划



###### 2.4.1软件版本规划

| 项目           | 版本                             | 备注                                           |
| -------------- | -------------------------------- | ---------------------------------------------- |
| HCI            | HCI5.3R1                         | 需要在5.3R1基础上打一个补丁包                  |
| 虚拟机操作系统 | Centos6.5                        | Linux版本推荐采用Centos6.5、Ubuntu14.04        |
| Hadoop         | 软件版本     Apache Hadoop 2.8.0 | 除了原生版本还可以选择发行版本：CDH、Hortwroks |
| JDK            | 版本     JDK1.8                  |                                                |
|                |                                  |                                                |



##### 2.5安装介质准备

可从Apache官网获取Hadoop安装压缩包，解压并配置好响应配置文件后使用。

### Hadoop大数据部署步骤  

##### 3.1Hadoop节点模板虚拟机部署

###### 3.1.1创建虚拟机

登录HCI Web控制台，在虚拟机-新增中，选择“创建全新虚拟机”(ACMP虚拟机)

1)名称：虚拟机名称可自定义，因为后续会部署Namenode虚拟机，因此为了后续部署利用方便，这里名称设置为Namenode。

2)存储位置：Hadoop所有虚拟机都是部署在本地存储中，这里需要选择一个本地存储位置。

3)CPU：启用NUMA调度

4)内存：默认配置32GB，可在后续hadoop部署中会根据实际群概况调整。

5)磁盘：最少配置200G的空间，主要用于存放日志。此阶段不需要勾选预分配存储空间，在后续hadoop部署阶段在勾选该选项。

6)光驱：加载操作系统安装镜像文件，例如：CentOS-6.5-x86_64-bin-DVD1.iso

7)网卡：**通常情况下建议大数据节点虚拟机部署在隔离网络中，这样可以减少大数据分析时，网络带宽占用过大导致对业务网络产生影响。**

8)高级配置：这里勾选“主机启动时，自动运行此虚拟机”、“标记为重要虚拟机”、“虚拟机异常时重启“



###### 3.1.2安装操作系统

虚拟机创建完成后，开机-安装系统，默认选择第一项：

 

跳过磁盘检测直接开始安装：

 

语言、键盘选择”English“、”U.S.English“，安装设备选择”Basic Storage 

Devices“：

安装设备确认框中点击”Yes,dicard my data“：

虚拟机Hostname配置为：hadoop.nn(方便后续利用)，网络暂不配置：

时区选择shanghai：

设置root密码后进行安装分区设置，在没有特殊安装需求情况下，默认选择”Use All Space“，在确认弹框中选择”Write Changes to disk“：

根据需求来选择安装Desktop或者Basic Server，如果有图形化操作需求就选择Desktop，如果没有就选择Basic Server。然后开始安装操作系统：



###### 3.1.3安装虚拟机性能优化工具

操作系统安装完成后，root用户登录系统，点击优化工具导航条上的”立即安装“，然后根据Linux安装的操作步骤进行虚拟机优化工具的安装并重启虚拟机

###### 3.1.4配置虚拟机网络

以Desktop图形化配置为例，从菜单找到网络配置项”Network Connections“打开配置界面

配置网络信息：

1)勾选”Connect Automactically“，开启自动连接

2)在IPv4 Settings的Method中选择Manual，然后点击Add进行IP地址配置

#####  3.1.5安装应用与环境变量配置

###### 3.1.5.1JDK安装与环境变量配置

​         上传JDK压缩包：将JDK压缩包上传到/opt目录下并解压：  

创建JAVA目录：在/usr目录下创建java目录，然后将解压后的JDK目录移动到/usr/java目录下：

配置JDK环境变量：在/etc/profile.d/目录下创建脚本”hadoopJDK.sh“

在vim编辑中，输入以下内容然后保存：

配置生效：执行命令：”source /etc/profile“命令使以上配置生效：

查看JAVA安装与配置：执行命令：”java -version“查看JDK安装与配置是否成功，版本与安装的一致则说明安装与配置成功



###### 3.1.5.2Hadoop应用安装与环境变量配置

上传Hadoop2.8.0压缩包：将Hadoop压缩包上传到/opt目录并解压

将解压后的hadoop目录移动到/home目录下

配置Hadoop环境变量：在/etc/profile.d目录下创建脚本”hadoopENV.sh“

在vim编辑中，输入以下内容然后保存，然后执行”source /etc/profile“让hadoop环境变量配置生效

Hadoop增加JDK环境配置信息：编辑/home/hadoop-2.8.0/etc/hadoop目录下的hadoop-env.sh配置文件

在hadoop-env.sh文件最后一行添加以下内容，然后保存

创建hadoop临时目录”/home/hdfs/tmp“和namenode目录”/home/hdfs/name“



###### 3.1.5.3Hadoop用户创建与授权

Hadoop用户是后续hadoop集群运维管理的用户，因为使用root这样的高权限用户进行hadoop运维工作会存在一定的安全风险。

创建用户hadoop（命令：useradd hadoop）和设置用户hadoop密码（命令：passwd hadoop）：

用户hadoop授权：给hadoop用户以及所在用户组（groups hadoop可以查看其用户组）授予管理hadoop应用相关目录的权利，包括：hadoop临时目录、namenode目录和安装目录

 



###### 3.1.5.4禁用防火墙和selinux

禁用防火墙：依次执行命令”service iptables stop“和”chkconfig iptables off“：

禁用selinux：编辑文件selinux配置文件”vim /etc/selinux/config“,将”SELINUX=enforcing“修改为”SELINUX=disabled“，然后保存

## 3.1.5.5系统内核参数优化

Hadoop在进行数据分析时，系统负载比较高，所以需要对内核参数、网络参数进行优化。

​          Linux系统最大进程数和最大文件打开数限制放开：编辑配置文件”vim /etc/security/limits.conf“,在文件最后增加以下参数内容后保存

​		系统网络参数优化：编辑配置文件”vim /etc/sysctl.conf“，在文件最后增加以下参数内容然后保存，然后执行命令：”sysctl -p“使配置生效



##### 3.2Hadoop节点部署

Hadoop节点模板虚拟机部署完毕后，通过**克隆**的方式部署Hadoop节点。

###### 3.2.1Hadoop节点虚拟机克隆

本次部署指南的Hadoop节点规划为：1*Namenode，1*SecondaryNamenode，6*Datanode，Namenode节点直接服用模板虚拟机，所以总共需要克隆7个虚拟机。

克隆SecondaryNamenode节点

  SecondaryNamenode节点顾名思义为辅助Namenode节点，节点保存了Namenode节点上的元数据镜像备份，当Namenode出现故障后，SecondaryNamenode节点可以承担Namenode的工作。因此SecondaryNamenode节点**不能与Namenode节点在同一个主机上，防止出现单点故障的情况**。

​		在克隆SecondaryNamenode节点时，指定的存储位置为其他主机的本地存储，不能与Namenode在同一本地存储中（Namenode存储位置为45_new,SecondaryNamenode存储位置指定为46_new）

克隆Datanode节点

  		克隆Datanode节点的原则是每台主机上的Datanode数量是一致的，这样可以让各个主机的压力负载更加均衡，同时数据分布也更加的均衡。因此按照第2章的Hadoop节点规划，总共6个Datanode节点，每台主机运行2个Datanode节点，所以按照顺序依次克隆2个Datanode节点到每台主机上（每个Datanode节点的克隆方法一致）



##### 3.2.2Hadoop节点初始化配置

Hadoop节点虚拟机克隆完毕后，需要进行集群初始化配置，配置内容为：主机名、网络、数据盘（Datanode需要）。

###### 3.2.2.1Hadoop节点主机名配置（hostname）

SecondaryNamenod节点主机名配置

  按照第2章Hadoop节点与IP规划中的SecondaryNamenode节点主机名规划进行配置，编辑配置文件：”/etc/sysconfig/network“,将”HOSTNAME“项修改为”hadoop.sn“,然后保存并reboot虚拟机

Datanode节点主机名配置

  修改方法与SecondaryNamenode节点一致，只是Datanode有多个节点，所以每个节点在修改时注意编号



###### 3.2.2.2Hadoop节点网络配置

SecondaryNamenode和Datanode节点修改网络配置的方法是一样的，所以下面以SecondaryNamenode网络配置修改作为操作示例。

由于SecondaryNamenode与Datanode都是由模板虚拟机克隆过来的，除了要修改网卡配置文件以外，还要重新生成网卡设备配置文件。

修改网卡配置文件

网卡配置文件需要修改的配置有：

1)IP地址：IPADDR

2)MAC地址：HWADDR

3)网卡UUID：UUID（可删除此项，建议保留）

编辑网卡配置文件：”/etc/sysconfig/network-script/ifcfg-eth0“，修改IPADDR、HWADDR、UUID三项后保存：

 

【说明】

1)IPADDR：按照第2章中的IP规划修改。

2)HWADDR：执行命令”ifconfig -a“查看上面的网卡信息，可以看到网卡的MAC地址（由于是克隆的虚拟机所以网卡编号不从0开始，可忽略）：

 

3)UUID：在操作系统中执行命令”nmcli con | sec -n ‘1,2p’“,就可以获取到当前已启用网卡的UUID（由于是克隆的虚拟机所以网卡编号不从0开始，可忽略）



重新生成网络设备文件

克隆的虚拟机需要重新生成一下网络设备文件，删除掉文件”/etc/udev/rules.d/70-persistent-net.rules“,然后reboot即可



###### 3.2.2.3Hadoop节点hosts文件配置

Hadoop节点间需要能够通过主机域名进行通信，所以需要把各个节点的主机名与IP地址对应关系写入到”/etc/hosts”文件中。只需要在Namenode上配置，然后将hosts文件拷贝到SecondaryNamenode和Datanode节点即可。

​         编辑“/etc/hosts”文件，将各个节点的主机名与地址对应关系写入

​         将已更新的hosts文件通过scp命令（scp /etc/hosts root@[节点主机名]：/etc）拷贝到SecondaryNamenode和各个Datanode节点的“/etc”目录下覆盖原来的hosts文件



###### 3.2.2.4Datanode节点数据盘配置

​         在HCI上部署Hadoop，Datanode的数据盘是采用物理磁盘裸透给虚拟机的方式，即直接将机械盘插到HCI主机上，不初始化为本地存储，datanode虚拟机通过加载物理盘的方式加载机械盘。  

Datanode机械盘添加策略

Datanode节点尽量采用**同构部署**，即：vCPU、内存、数据盘数量、数据盘大小保持一致，通常一台HCI主机，我们是建议运行2个Datanode节点，所以用于每台主机用于Datanode的机械盘要为**偶数**。

以本次指南为例，用于Datanode的机械盘为6，那么每个Datanode节点挂载3块机械盘。

Datanode添加数据盘

1)以hadoop.dn1节点为例，编辑虚拟机，选择添加添加硬件-硬盘

2)选择物理盘选项，然后在未使用磁盘中，依次添加可用的磁盘。这里需要注意，第一个添加的节点不要把所有磁盘全部添加，只添加1/2的未使用磁盘即可，剩下的磁盘留给主机上的另一个datanode节点

Datanode格式化数据盘

1)启动hadoop.dn1节点虚拟机，并以root用户登录

2)对添加的数据盘进行分区：

通过命令“lsblk -d”查看刚才添加的磁盘信息，通过物理盘加载的方式挂载的磁盘，盘符都是以“sd”开头，所以下图中的“sda、sdb、sdc”就是刚才添加的物理磁盘：

 

如果采用的是4T的磁盘，所以需要采用GPT分区，如果2T或以下的磁盘分区可以采用MBR分区，以下为4T盘分区步骤。以sda为例，按照下图的步骤进行磁盘分区操作：

 

   通过命令”lsblk“查看磁盘，可以看到sda磁盘下多了一个sda1的磁盘分区：

 

【说明】

①　主要的分区操作命令是parted /dev/sda(进入GPT分区界面)，mklabel gpt（创建gpt标签），mkpart primary 0 -1(把磁盘所有空间用于一个分区)，其他的都是告警信息，输入yes或者ignore即可

②　所有Datanode的所有数据盘都按照此操作进行磁盘分区。依次对sda、sdb、sdc进行分区。

3) 磁盘分区完成后需要对磁盘分区进行格式化，格式化命令为：”mkfs.ext4 /dev/sda1“。并依次对sda1、sdb1、sdc1进行格式化:



Datanode节点挂载数据盘

1)创建磁盘挂载目录。依次为/dev/sda1、/dev/sdb1、/dev/sdc1创建挂载目录，执行命令”mkdir -p /hadoop/sdx1“(x为对应的磁盘编号)

2)将磁盘依次挂载到指定目录。执行命令”mount /dev/sdx1 /hadoop/sdx1“(x为对应的磁盘编号)

3)将磁盘挂载对应关系写入到开启磁盘挂载配置文件中。编辑”/etc/fstab“,按照红框中的格式输入磁盘挂载对应信息

【说明】

  如果采用的是2T或者2T以下磁盘，分区方法如下：

1)首先lsblk查看磁盘加载状态

2)以vde为例，执行“fdisk /dev/vde”

3)输入“n”开始分区配置

4)输入“p”设置第一个主分区

5)输入“1”设置主分区编号为1

6)输入“1024”设置起始柱面：建议设置为1024

7)磁盘只分一个分区，所有剩余的全部给第一分区即可，所以直接按回车

8)输入“w”将分区配置写入，即完成磁盘分区配置：

9)后续的磁盘格式化，参考4T磁盘方法。

Datanode数据盘目录用户授权

  需要将Datanode的数据盘目录授权给hadoop用户，执行命令”chown -R hadoop:hadoop /hadoop/sdx1“(x为对应目录编号)



##### 3.2.2.5节点间SSH无密码登录配置

需要指定hadoop用户（非root用户，第3章中创建的hadoop集群管理用户）在hadoop的指定节点间配置SSH无密码登录：

1)Namendoe、SecondaryNamenode与Datanode之间配置SSH无密码登录：远程执行命令需求。

2)Namenode与SecondaryNamenode之间配置SSH无密码登录：元数据同步需求。

配置Namenode到各个Datanode节点间的SSH无密码登录

1)hadoop用户登录到Namenode节点，执行命令“ssh-keygen -t rsa”一路回车生成密钥

2)再执行命令“ssh-copy-id -i ~/.ssh/id_rsa.pub 主机名”分别把密钥拷贝到hadoop.nn(本节点)和所有Datanode节点上即完成SSH无密码登录配置，这个过程中需要输入“yes”确认和输入hadoop用户的密码验证。参考以下实例进行：

3)验证SSH无密码登录配置是否成功。在Namenode节点ssh任意一个Datanode节点，不在需要输入密码：

配置SecondaryNamenode到各个Datanode间的SSH无密码登录

配置方法同Namenode到各个Datanode间的SSH无密码登录方法。

配置Namenode到SecondaryNamenode的SSH无密码登录

直接通过命令“ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop.sn”拷贝密钥到SecondaryNamenode节点即可。

配置SecondaryNamnode到Namenode的SSH无密码登录

  直接通过命令“ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop.nn”拷贝密钥到Namenode节点即可。

##### 3.3Hadoop集群配置

###### 3.3.1Hadoop集群配置文件配置

Hadoop集群所有相关的配置文件都在”/home/hadoop-2.8.0/etc/hadoop“目录下，由于所有节点是通过模板虚拟机统一部署，所以可以在Namenode节点上完成所有配置文件的配置后，再将相关配置文件拷贝到SecondaryNamenode和各个Datanode节点即可。

【说明】

**如果是采用Hadoop节点异构的部署方式，就需要在各个节点对配置文件进行本地化调整。**

###### 3.3.1.1masters文件配置

Hadoop默认配置是把Namenode与SecondaryNamenode部署在同一个节点上，要分开配置需要创建并配置masters文件（如果已有masters文件不需要重新创建）然后把Namenode和SeconNamenode信息添加到文件中。

在“/home/hadoop-2.8.0/etc/hadoop”目录下创建“masters”文件

vim编辑“masters”文件，将Namenode和SecondaryNamenode的主机名写入，每个主机名占一行然后保存

###### 3.3.1.2slaves文件配置

slaves文件保存的是Datanode节点的信息，默认下这个文件是存在于/home/hadoop-2.8.0/etc/hadoop”目录，如果没有需要手动创建。

​         vim编辑“slaves”文件，首次部署hadoop该文件中会有一个默认值“localhost”，删除掉该行，然后增加所有Datanode的主机信息，每个主机信息占一行然后保存



###### 3.3.1.3core-site.xml文件配置

**core-site.xml文件为hadoop的核心配置文件，这里配置的是整个hadoop集群的基础配置信息**。编辑“/home/hadoop-2.8.0/etc/hadoop/core-site.xml”文件，按照以下表格内容添加到文件中后保存：

| 参数名：name        | 参数值：value         | 参数描述                                       |
| ------------------- | --------------------- | ---------------------------------------------- |
| fs.default.name     | hdfs://hadoop.nn:9000 | 集群名称，格式：hdfs://[Namenode主机名]:[端口] |
| io.file.buffer.size | 131072                | 流文件缓冲区大小，单位为bytes                  |
| hadoop.tmp.dir      | /home/hdfs/tmp        | hadoop临时文件目录                             |
|                     |                       |                                                |

###### 3.3.1.4hdfs-site.xml文件配置

**hdfs-site.xml文件主要用来设置与HDFS文件系统相关的配置信息**。编辑“/home/hadoop-2.8.0/etc/hadoop/hdfs-site.xml”文件，按照以下表格内容添加到文件中后保存：

| 参数名：name                              | 参数值：value                            | 参数描述                                                     |
| ----------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| dfs.http.address                          | hadoop.nn:50070                          | Namenode主机名以及web管理端口配置                            |
| dfs.namenode.secondary.http-address       | hadoop.sn:50090                          | SecondaryNamenode主机名以及web管理端口配置                   |
| dfs.namenode.name.dir                     | file:/home/hdfs/name                     | **Namenode的工作目录**                                       |
| dfs.namenode.checkpoint.check.period      | 60                                       | 检查触发checkpoint的条件是否满足的频率，单位：秒             |
| dfs.namenode.checkpoint.dir               | file:/home/hdfs/tmp/dfs/namesecondary    | **SecondaryName的工作目录**                                  |
| dfs.namenode.checkpoint.edits.dir         | ${dfs.namenode.checkpoint.dir}           | SecondaryName每次checkpoint从Namendoe拷贝的HDFS元数据文件fsimage存放目录，默认与工作目录相同 |
| dfs.namenode.checkpoint.period            | 3600                                     | 两个checkpoint之间的时间间隔，单位：秒                       |
| dfs.namenode.checkpoint.txns              | 100 0000                                 | 两个checkpoint之间的最大的操作记录（与上面条件任一满足即触发checkpoint） |
| dfs.blocksize                             | 256m                                     | HDFS文件系统块大小，单位是：MBytes                           |
| dfs.replication                           | 3                                        | HDFS文件系统副本数                                           |
| dfs.datanode.data.dir                     | /hadoop/sda1, /hadoop/sdb1, /hadoop/sdc1 | Datanode数据存放目录，多个目录通过”，“隔开                   |
| dfs.datanode.max.transfer.threads         | 8192                                     | DataNode间传输block数据的最大线程数                          |
| dfs.datanode.balance.bandwidthPerSec      | 10485760                                 | DataNode用于balancer的带宽，单位：Bytes/sec                  |
| dfs.datanode.balance.max.concurrent.moves | 50                                       | DataNode上同时用于balance待移动block的最大线程个数           |

 

###### 3.3.1.5yarn-site.xml文件配置

**yarn是hadoop 2.x才有的新的资源调度架构**，所以此配置文件用于对资源调度方面的参数进行配置，“/home/hadoop-2.8.0/etc/hadoop/yarn-site.xml”文件，按照以下表格内容添加到文件中后保存

| 参数名：name                                  | 参数值：value                             | 参数描述                                                     |
| --------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------ |
| yarn.nodemanager.aux-services                 | mapreduce_shuffle nodemanager             | 附属服务，需要配置为mapreduce_shuffle才能运行mapreduce程序   |
| yarn.resourcemanager.address                  | hadoop.nn:8032                            | 客户端通过该地址向RM提交应用程序，杀死应用程序等,格式：[Namnode主机名]:[端口],下同 |
| yarn.resourcemanager.scheduler.address        | hadoop.nn:8030                            | ApplicationMaster通过该地址向RM申请资源、释放资源等          |
| yarn.resourcemanager.resource-tracker.address | hadoop.nn:8031                            | NodeManager通过该地址向RM汇报心跳，领取任务等                |
| yarn.resourcemanager.admin.address            | hadoop.nn:8033                            | 管理员通过该地址向RM发送管理命令等。                         |
| yarn.resourcemanager.webapp.address           | hadoop.nn:8088                            | ResourceManager对外web ui地址。用户可通过该地址在浏览器中查看集群各类信息。 |
| yarn.nodemanager.resource.cpu-vcores          | 24                                        | NodeManager总的可用虚拟CPU个数                               |
| yarn.nodemanager.resource.memory-mb           | 45056                                     | NodeManager总的可用物理内存                                  |
| yarn.nodemanager.local-dirs                   | /hadoop/sda1,  /hadoop/sdb1, /hadoop/sdc1 | Mapreduce中间结果存放位置，默认设置为datanode数据目录，多个目录之间用“，”隔开 |

 

###### 3.3.1.6mapred-site.xml文件配置

**在运行mapreduce程序时，需要从mapred-site.xml中读取相关应用配置信息，首次配置需要拷贝一份配置文件**，进入目录“/home/hadoop-2.8.0/etc/hadoop”执行“cp mapred-site.xml.template mapred-site.xml”，然后在编辑mapreduce-site.xml文件，按照以下表格内容添加到文件中后保存：

| 参数名：name                            | 参数值：value                              | 参数描述                                                     |
| --------------------------------------- | ------------------------------------------ | ------------------------------------------------------------ |
| mapreduce.framework.name                | yarn                                       | **指定使用yarn来运行mapreduce程序**                          |
| mapreduce.jobhistory.address            | hadoop.nn:10020                            | 已经运行完的Mapreduce作业记录信息存放服务器，默认设置为Namenode服务器 |
| mapreduce.jobhistory.webapp.address     | hadoop.nn:19888                            | 通过web ui访问查看mapreduce作业信息的路径配置                |
| mapreduce.task.io.sort.mb               | 400                                        | Map task缓冲区所占内存大小，单位：MBytes                     |
| mapreduce.map.memory.mb                 | 4096                                       | 每个Map task最大可申请的内存大小，单位：MBytes               |
| mapreduce.reduce.memory.mb              | 8192                                       | 每个reduce task最大可申请的内存大小，单位：MBytes            |
| mapreduce.map.cpu.vcores                | 1                                          | 每个map task最大使用的cpu vcore数量                          |
| mapreduce.reduce.cpu.vcores             | 2                                          | 每个reduce task最大使用的cpu vcore数量                       |
| mapreduce.job.heap.memory-mb.ratio      | 0.8                                        |                                                              |
| mapred.child.java.opts                  | -Xmx2048m                                  | JVM堆大小配置                                                |
| mapreduce.map.speculative               | false                                      | map task预测执行关闭                                         |
| mapreduce.reduce.speculative            | false                                      | reduce task预测执行关闭                                      |
| mapreduce.reduce.shuffle.parallelcopies | 10                                         | *re*uduce shuffle阶段同时从map task节点提取数据的线程数      |
| mapreduce.task.io.sort.factor           | 40                                         | reduce Task中合并小文件时，一次合并的文件数据                |
| mapreduce.map.output.compress           | true                                       | 启用map输出压缩                                              |
| mapreduce.map.output.compress.codec     | org.apache.hadoop.io.compress.DefaultCodec | 配置默认的压缩算法                                           |



#### 3.3.1.7Hadoop集群机架感知配置（非常重要）

​         机架感知是hadoop集群中非常重要的配置，通常HDFS副本数都是配置2副本或以上，在实际用户场景中最多的是HDFS 2副本和3副本混合的情况。  所以**当HDFS只要存在一个可用的副本，那么大数据分析处理就能够正常进行**。为了防止一份数据的所有副本的对应datanode节点在同一个机架上，当这个机架出现故障导致数据丢失，hadoop提供一个机架感知的功能，在插入数据到HDFS时，将一份数据的副本分散到不同机架上，当任意一个机架出现问题时，总会有另一个机架的副本提供数据支持。机架感知针对2副本和3副本有两种不同的策略：

1)2副本：一份数据的2个副本分散到两个不同机架的datanode节点。

2)3副本：一份数据的两个副本放在同一个机架的不同datanode节点，第三个副本放到另一个机架的datanode节点。这样可以保证数据安全，**同时在分析数据时尽量让任务尽量落在副本在统一机架的datanode上，这样可以保证分析的效率，同时降低了跨机架取数据对网络带宽的压力**。

​         **在HCI部署Hadoop，机架感知的配置与传统的物理服务器上部署hadoop不同**  ，在HCI上，一个主机部署多个datanode节点，所以HCI主机就相当于物理环境的一个机架。因此在配置机架感知时，将同一HCI主机上运行的datanode规划到同一机架中。

配置机架感知需要配置三个部分：机架感知检测脚本、机架感知配置文件、机架感知配置参数。

​         机架感知检测脚本创建和编辑  

在${HADOOP_CONF}目录“/home/hadoop-2.8.0/etc/hadoop”目录下通过vim创建机架感知检测脚本“RackAware.py”，在vim编辑界面中输入一下内容然后保存：

​         通过命令“chmod 750 RackAware.py”给脚本授权。  

【说明】

1)”hadoop.dnx“：”rackx“，hadoop.dnx为datanode节点主机名，rackx为机架名，这个可以自定义。

2)“192.168.1.x”为datanode节点IP地址，因为在namenode上需要通过主机名去解析，在datnode节点上需要通过IP地址去解析，所以这里主机名和IP地址的配置信息都需要添加进去。

3)配置原则：一个HCI主机为一个机架rack，相同主机上运行的datanode配置在一个机架编号内。

4)只需要配置datanode主机名和IP地址对应机架编号部分，后面的不需要修改。

机架感知配置参数

机架感知除了检测到脚本和配置文件以外，还需要在编辑core-site.xml 文件，增加以下以下参数内容：

| 参数名：name              | 参数值：value                             | 参数描述                   |
| ------------------------- | ----------------------------------------- | -------------------------- |
| topology.script.file.name | /opt/hadoop-2.8.0/etc/hadoop/RackAware.py | 配置机架感知检测的脚本路径 |
|                           |                                           |                            |



###### 3.3.1.8Hadoop集群配置文件同步

进入Namenode的${HADOOP_CONF}目录，将所有配置文件通过scp从Namenode依次拷贝到SecondaryNamenode和Datanode节点的${HADOOP_CONF}目录下，即“/home/hadoop-2.8.0/etc/hadoop”目录下，scp命令为：

“scp masters slaves core-site.xml hdfs-site.xml yarn-site.xml mapred-site.xml RackAware.py hadoop@hadoop.sn:`pwd`”

【说明】

 如果HCI主机为异构部署，配置文件拷贝到各个节点后，磁盘、内存、CPU配置参数需要按照当前主机配置进行调整。



###### 3.3.2Hadoop集群HDFS文件系统初始化

###### 3.3.2.1HDFS文件系统初始化

以hadoop用户登录Namenode节点（或者root登录后执行命令”su - hadoop“切换到hadoop用户）

执行命令”hdfs namenode -format“,初始化HDFS系统

​         **不能有报错信息，只能是INFO信息,如果有报错说明存在配置错误的情况**

【说明】

初始化和后续执行hadoop mapreduce程序过程中，会出现以下告警信息：

”WARN util.NativeCodeLoader: Unable to load naive-hadoop library for your platform... using builtin-java classes where applicable“，这个不影响，可以忽略。



**3.3.2.2Hadoop集群启动与状态检查**

启动hadoop集群：以hadoop用户登录Namenode节点（或者root用户登录后su - hadoop切换到hadoop用户），进入”/home/hadoop-2.8.0/sbin“目录，执行脚本”./start-all.sh“启动hadoop集群。可以看到Namenode、SecondaryNamenode和所有Datanode节点都被启动：

 

检查所有节点对应的进程是否启动成功：

1)Namenode：Namenode节点上运行的进程包括：Namenode、ResourceManager，通过hadoop用户登录Namenode节点，然后执行命令jps，查看对应进程启动情况，能够显示出来就说明启动成功：

2)SecondaryNamenode：SecondaryNamenode节点上运行的进程为：SecondaryNameNode，查看方法相同，执行jps命令查看：

3)Datanode：Datanode节点上运行的进程包括：DataNode、NodeManager，执行命令jps查看：

 

  所有节点上对那个的进程都起来后，说明hadoop集群已经正常启动，并且所有节点的相关目录状态和节点信息配置正确

检查hadoop节点和文件系统状态

  以hadoop用户登录Namenode节点，执行命令”hadoop dfsadmin -report“能够查看到hadoop整个集群的节点状态和HDFS文件系统信息：

1)查看Live datanode，显示的数量与实际的数量一致，说明hadoop集群所有的datanode状态正常。

2)查看总的和各个节点的DFS Remaining，容量与实际一致，说明HDFS文件系统正常



###### 3.3.2.3Hadoop机架感知配置状态检查

以hadoop用户登录Namenode节点，执行命令”hadoop dfsadmin -printToplogy“查看机架感知状态：

 

   所有datanode节点按照配置脚本中的机架rack进行归类显示，即说明hadoop集群机架感知已经配置生效。

 

 
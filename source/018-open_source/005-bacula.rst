bacula
#############
开源备份软件


介绍
=======

Bacula是一个开源网络备份解决方案，允许您创建备份和执行计算机系统的数据恢复。它非常灵活和健壮，这使得它，虽然配置稍微麻烦，适合在许多情况下的备份。备份系统是在大多数服务器基础架构的重要组成部分 ，从数据丢失恢复往往是灾难恢复计划的重要组成部分。 在本教程中，我们将向您展示如何在CentOS 7服务器上安装和配置Bacula的服务器组件。我们将配置Bacula执行每周作业，创建本地备份（即其自己的主机的备份）。这本身并不是Bacula的特别引人注目的用途，但它将为您创建其他服务器（即备份客户端）的备份提供一个良好的起点。本系列的下一个教程将介绍如何通过安装和配置Bacula客户端以及配置Bacula服务器来创建其他远程服务器的备份。 如果您想使用Ubuntu 14.04代替，请点击此链接： 如何在Ubuntu 14.04安装Bacula的服务器 。

本次安装中，备份服务器主机名是k8s1.alv.pub   客户端是k8s2.alv.pub ,都可以通过这两个域名访问到对应的ip。


安装bacula
==================

bacula下载地址：http://blog.bacula.org/source-download-center/



下载软件
---------------

.. code-block:: bash

    wget http://soft.alv.pub/linux/bacula-9.4.1.tar.gz

解压软件
----------

::

    tar xf bacula-9.4.1.tar.gz

安装依赖包
---------------

::

    yum install gcc gcc-c++ mysql-devel -y
    cd bacula-9.4.1/


检查编译环境
------------------

::

    cd bacula-9.4.1/
     ./configure --prefix=/usr/local/bacula --sbindir=/usr/local/bacula/sbin --sysconfdir=/usr/local/bacula/etc --enable-smartalloc --with-working-dir=/usr/local/bacula/bin/working --with-subsys-dir=/usr/local/bacula/bin/working --with-pid-dir=/usr/local/bacula/bin/working --with-mysql



开始编译
--------------

这里指定了路径为/usr/local/bacula，默认情况下,bacula 的安装路径为/etc/bacula.


::

    make && make install
    make install-autostart

设置环境变量
-------------------


::

    $ vim /etc/profile
    export PATH=$PATH:/usr/local/bacula/sbin
    $ source /etc/profile

初始化Mysql数据库
========================


::

    yum install mariadb-server -y
    systemctl restart mariadb
    systemctl enable mariadb
    cd /usr/local/bacula/etc
     ./grant_mysql_privileges
    ./create_mysql_database
    ./make_mysql_tables



bacula实例配置
==================


Console端的配置
---------------------------
想要直接使用的话，默认配置不用做修改，下面的内容做示例解释。

::

    $ vim bconsole.conf // Console端的配置文件

    Director

    {

           Name = f10-64-build-dir  #控制端名称，在下面的bacula-dir.conf和bacula-sd.conf

           #文件中会陆续的被引用

          DIRport = 9101    #控制端服务端口

          address = 10.0.253.117  #控制端服务器IP地址

          Password = "ouDao0SGXx/F+Tx4YygkK4so0l/ieqGJIkQ5DMsTQh6t"

           #控制端密码文件

    }

::

    $ vim bacula-dir.conf //Director端的配置文件

    Director {                            #定义bacula的全局配置

          Name = f10-64-build-dir

          DIRport = 9101                 #定义Director的监听端口

          QueryFile = "/usr/local/bacula/etc/query.sql"

          WorkingDirectory = "/usr/local/bacula/var/bacula/working"

          PidDirectory = "/var/run"

          Maximum Concurrent Jobs = 1    #定义一次能处理的最大并发数



         #验证密码，这个密码必须与bconsole.conf文件中对应的Director逻辑段密码相同

          Password = "ouDao0SGXx/F+Tx4YygkK4so0l/ieqGJIkQ5DMsTQh6t"



          #定义日志输出方式，“Daemon”在下面的Messages逻辑段中进行了定义

          Messages = Daemon

    }



    Job {        #自定义一个备份任务

           Name = "Client1"  #备份任务名称

          Client = dbfd   #指定要备份的客户端主机，“dbfd”在后面Client逻辑段中

           #进行定义

          Level = Incremental      #定义备份的级别，Incremental为增量备份。Level的取值#可为Full（完全备份）、Incremental（增量备份）和Differential（差异备份），如果第一#次没做完全备份，则先进行完全备份后再执行Incremental

          Type = Backup                  #定义Job的类型，“backup”为备份任务，可选

           #的类型还有restore"恢复"和verify"验证"等

          FileSet = dbfs     #指定要备份的客户端数据，“dbfs”在后面FileSet

           #逻辑段中进行定义

          Schedule = dbscd    #指定这个备份任务的执行时间策略，“dbscd”在

           #后面的Schedule逻辑段中进行了定义

          Storage = dbsd     #指定备份数据的存储路径与介质，“dbsd” 在后

           #面的Storage逻辑段中进行定义

          Messages = Standard

          Pool = dbpool     #指定备份使用的pool属性，“dbpool”在后面的

           # Pool逻辑段中进行定义。

          Write Bootstrap = "/usr/local/bacula/var/bacula/working/Client2.bsr" #指定备份的引导信息路径

    }





    Job {         #定义一个名为Client的差异备份的任务

          Name = "Client"

          Type = Backup

          FileSet = dbfs

          Schedule = dbscd

          Storage = dbsd

          Messages = Standard

          Pool = dbpool

          Client = dbfd

          Level = Differential      #指定备份级别为差异备份

          Write Bootstrap = "/usr/local/bacula/var/bacula/working/Client1.bsr"

    }





    Job {        #定义一个名为BackupCatalog的完全备份任务

          Name = "BackupCatalog"

          Type = Backup

          Level = Full        #指定备份级别为完全备份

          Client = dbfd

          FileSet="dbfs"

          Schedule = "dbscd"

          Pool = dbpool

          Storage = dbsd

          Messages = Standard

          RunBeforeJob = "/usr/local/bacula/etc/make_catalog_backup bacula bacula"

          RunAfterJob  = "/usr/local/bacula/etc/delete_catalog_backup"

          Write Bootstrap = "/usr/local/var/bacula/working/BackupCatalog.bsr"

    }





    Job {           #定义一个还原任务

          Name = "RestoreFiles"

          Type = Restore       #定义Job的类型为“Restore ”，即恢复数据

          Client=dbfd

          FileSet=dbfs

          Storage = dbsd

          Pool = dbpool

          Messages = Standard

          Where = /tmp/bacula-restores  #指定默认恢复数据到这个路径

    }





    FileSet {  #定义一个名为dbfs的备份资源，也就是指定需要备份哪些数据，需要排除哪

    #些数据等，可以指定多个FileSet

          Name = dbfs

          Include {

               Options {

              signature = MD5; Compression=GZIP; }   #表示使用MD5签名并压缩

               File = /cws3            #指定客户端FD需要备份的文件目录

     }



    Exclude {    #通过Exclude排除不需要备份的文件或者目录，可根据具体情况修改

               File = /usr/local/bacula/var/bacula/working

               File = /tmp

               File = /proc

               File = /tmp

               File = /.journal

               File = /.fsck

     }

    }



    Schedule {        #定义一个名为dbscd的备份任务调度策略

          Name = dbscd

          Run = Full 1st sun at 23:05  #第一周的周日晚23:05分进行完全备份

          Run = Differential 2nd-5th sun at 23:05 #第2~5周的周日晚23:05进行差异备份

          Run = Incremental mon-sat at 23:05  #所有周一至周六晚23:05分进行增量备份

    }





    FileSet {

          Name = "Catalog"

          Include {

               Options {

              signature = MD5

               }

               File = /usr/local/bacula/var/bacula/working/bacula.sql

      }

    }





    Client {        #Client用来定义备份哪个客户端FD的数据

          Name = dbfd  #Clinet的名称，可以在前面的Job中调用

          Address = 10.0.253.118    #要备份的客户端FD主机的IP地址

          FDPort = 9102      #与客户端FD通信的端口

          Catalog = MyCatalog     #使用哪个数据库存储信息，“MyCatalog”在后面

           #的MyCatalog逻辑段中进行定义

          Password = "ouDao0SGXx/F+Tx4YygkK4so0l/ieqGJIkQ5DMsTQh6t"  #Director端与客户端FD

           #的验证密码，这个值必须与客户端FD配置文件bacula-fd.conf中密码相同

          File Retention = 30 days    #指定保存在数据库中的记录多久循环一次，这里是30天，只

           #影响数据库中的记录不影响备份的文件

          Job Retention = 6 months  #指定Job的保持周期，应该大于File Retention指定的值

          AutoPrune = yes          #当达到指定的保持周期时，是否自动删除数据库中的记录，

           #yes表示自动清除过期的Job

    }



    Client {

          Name = dbfd1

          Address = 10.0.253.118

          FDPort = 9102

          Catalog = MyCatalog

          Password = "Wr8lj3q51PgZ21U2FSaTXICYhLmQkT1XhHbm8a6/j8Bz"

          File Retention = 30 days

          Job Retention = 6 months

          AutoPrune = yes

    }





    Storage {      # Storage用来定义将客户端的数据备份到哪个存储设备上

          Name = dbsd

          Address = 10.0.253.117  #指定存储端SD的IP地址

          SDPort = 9103    #指定存储端SD通信的端口

          Password = "ouDao0SGXx/F+Tx4YygkK4so0l/ieqGJIkQ5DMsTQh6t"  #Director端与存储端

           #SD的验证密码，这个值必须与存储端SD配置文件bacula-sd.conf中Director逻辑段密码

           #相同

          Device = dbdev #指定数据备份的存储介质，必须与存储端（这里是10.0.253.117）

           #的bacula-sd.conf配置文件中的“Device” 逻辑段的“Name”项名称相同

          Media Type = File #指定存储介质的类别，必须与存储端SD（这里是10.0.253.117）

           #的bacula-sd.conf配置文件中的“Device” 逻辑段的“Media Type”项名称相同



    }



    Catalog {         # Catalog逻辑段用来定义关于日志和数据库设定

          ame = MyCatalog

          dbname = "bacula"; dbuser = "bacula"; dbpassword = ""   #指定库名、用户名和密码

    }



    Messages { # Messages逻辑段用来设定Director端如何保存日志，以及日志的保存格式，

           #可以将日志信息发送到管理员邮箱，前提是必须开启sendmail服务

          Name = Standard

          mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: %t %e of %c %l\" %r"

          operatorcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: Intervention needed for %j\" %r"

          mail = dba.gao@gmail.com = all, !skipped

          operator = exitgogo@126.com = mount

          console = all, !skipped, !saved

          append = "/usr/local/bacula/log/bacula.log" = all, !skipped   #定义bacula的运行日志

          append ="/usr/local/bacula/log/bacula.err.log" = error,warning, fatal #定义bacula的错误日志

          catalog = all

    }



    Messages { #定义了一个名为Daemon的Messages逻辑段，“Daemon”已经

           #在前面进行了引用

          Name = Daemon

          mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula daemon message\" %r"

          mail = exitgogo@126.com = all, !skipped

          console = all, !skipped, !saved

          append = "/usr/local/bacula/log/bacula_demo.log" = all, !skipped

    }





    Pool {  #定义供Job任务使用的池属性信息，例如，设定备份文件过期时间、

           #是否覆盖过期的备份数据、是否自动清除过期备份等

          Name = dbpool

          Pool Type = Backup

          Recycle = yes                   #重复使用

          AutoPrune = yes                #表示自动清除过期备份文件

          Volume Retention = 7 days        #指定备份文件保留的时间

          Label Format ="db-${Year}-${Month:p/2/0/r}-${Day:p/2/0/r}-id${JobId}" #设定备份文件的

           #命名格式，这个设定格式会产生的命名文件为：db-2010-04-18-id139

          Maximum Volumes = 7  #设置最多保存多少个备份文件

          Recycle Current Volume = yes #表示可以使用最近过期的备份文件来存储新备份

          Maximum Volume Jobs = 1  #表示每次执行备份任务创建一个备份文件

    }



    Console {      #限定Console利用tray-monitor获得Director的状态信息

          Name = f10-64-build-mon

          Password = "RSQy3sRjak3ktZ8Hr07gc728VkZHBr0QCjOC5x3pXEap"

          CommandACL = status, .status

    }





配置bacula的SD:
-----------------------


::

    $ vim bacula-sd.conf//服务器端的配置文件

           Storage {                 #定义存储，本例中是f10-64-build-sd

          Name = f10-64-build-sd #定义存储名称

          SDPort = 9103           #监听端口

          WorkingDirectory = "/usr/local/bacula/var/bacula/working"

          Pid Directory = "/var/run"

          Maximum Concurrent Jobs = 20

    }



    Director {        #定义一个控制StorageDaemon的Director

          Name = f10-64-build-dir     #这里的“Name”值必须和Director端配置文件

           #bacula-dir.conf中Director逻辑段名称相同

          Password = "ouDao0SGXx/F+Tx4YygkK4so0l/ieqGJIkQ5DMsTQh6t"   #这里的“Password”值

           #必须和Director端配置文件bacula-dir.conf中Storage逻辑段密码相同

    }



    Director {      #定义一个监控端的Director

          Name = f10-64-build-mon    #这里的“Name”值必须和Director端配置文件

           #bacula-dir.conf中Console逻辑段名称相同

          Password = "RSQy3sRjak3ktZ8Hr07gc728VkZHBr0QCjOC5x3pXEap"   #这里的“Password”

           #值必须和Director端配置文件bacula-dir.conf中Console逻辑段密码相同

          Monitor = yes

    }



    Device {       #定义Device

          Name = dbdev    #定义Device的名称，这个名称在Director端配置文件

           #bacula-dir.conf中的Storage逻辑段Device项中被引用

          Media Type = File   #指定存储介质的类型，File表示使用文件系统存储

          Archive Device = /webdata  #Archive Device用来指定备份存储的介质，可以

           #是cd、dvd、tap等，这里是将备份的文件保存的/webdata目录下

           LabelMedia = yes;            #通过Label命令来建立卷文件

          Random Access = yes;   #设置是否采用随机访问存储介质，这里选择yes

          AutomaticMount = yes;  #表示当存储设备打开时，是否自动使用它，这选择yes

          RemovableMedia = no;  #是否支持可移动的设备，如tap或cd，这里选择no

          AlwaysOpen = no;   #是否确保tap设备总是可用，这里没有使用tap设备，

           #因此设置为no

    }



    Messages {        #为存储端SD定义一个日志或消息处理机制

          Name = Standard

          director = f10-64-build-dir = all

    }



配置bacula的FD端
----------------------


::

    $ vim fd.conf //客户端的配置文件

    Director {      #定义一个允许连接FD的控制端

          Name = f10-64-build-dir  #这里的“Name”值必须和Director端配置文件
           #bacula-dir.conf中Director逻辑段名称相同
          Password = "ouDao0SGXx/F+Tx4YygkK4so0l/ieqGJIkQ5DMsTQh6t"  #这里的“Password”
           #值必须和Director端配置文件bacula-dir.conf中Client逻辑段密码相同

    }



    Director {      #定义一个允许连接FD的监控端
          Name = f10-64-build-mon
          Password = "RSQy3sRjak3ktZ8Hr07gc728VkZHBr0QCjOC5x3pXEap"
          Monitor = yes

    }



    FileDaemon {                #定义一个FD端
          Name = localhost.localdomain-fd
          FDport = 9102                  #监控端口
          WorkingDirectory = /usr/local/bacula/var/bacula/working
          Pid Directory = /var/run
          Maximum Concurrent Jobs = 20   #定义一次能处理的并发作业数
    }


    Messages {      #定义一个用于FD端的Messages
          Name = Standard
          director = localhost.localdomain-dir = all, !skipped, !restored
    }





服务器端的启动
--------------------

::

    $ /usr/local/bacula/sbin/bacula
    {start|stop|restart|status}


也可以通过分别管理bacula各个配置端的方式，依次启动或者关闭每个服务：


::

    /usr/local/bacula/etc/bacula-ctl-dir  {start|stop|restart|status}

    /usr/local/bacula/etc/bacula-ctl-sd  {start|stop|restart|status}

    /usr/local/bacula/etc/bacula-ctl-fd  {start|stop|restart|status}



客户端的启动：
------------------


::

    /usr/local/bacula/sbin/bacula start
    Starting the Bacula File daemon

管理客户端FD的服务，也可以通过以下方式完成：



::

    /usr/local/bacula/sbin/bacula {start|stop|restart|status}
    /usr/local/bacula/etc/bacula-ctl-fd  {start|stop|restart|status}



简单实例运行：

备份恢复：

::

    $ /usr/local/bacula/sbin/bconsole
    Connecting to Director 10.0.253.117:9101

    1000 OK: f10-64-build-dir Version: 3.0.2 (18 July 2009)

    Enter a period to cancel a command

    *run



客户端独立安装配置
========================


安装软件

::

    yum install gcc gcc-c++ mysql-devel -y
    wget http://soft.alv.pub/linux/bacula-9.4.1.tar.gz
    tar xf bacula-9.4.1.tar.gz
    cd bacula-9.4.1/
    ./configure --enable-client-only --prefix=/usr/local/bacula
    make && make install && make install-autostart

配置客户端


这里我们要将配置文件里前面两个Director里的name改成服务器上的配置，


::

    $ vim /usr/local/bacula/etc/bacula-fd.conf
    Director {
      Name = k8s1.alv.pub-dir
      Password = "F/FnmGaHpEeoFdSlAfMgmwchYh/eG/tUBd7IFcY1K1EZ"
    }

    #
    # Restricted Director, used by tray-monitor to get the
    #   status of the file daemon
    #
    Director {
      Name = k8s1.alv.pub-mon
      Password = "0h2sMW+pinqWRSMQ3i5xcYQKoMK8RCSEHQtRTesfXjhS"
      Monitor = yes
    }


启动服务

::

    systemctl start bacula-fd



客户端准备好要用于备份的目录。

::

    [root@k8s2 ~]# mkdir -p /data/dir{1..5}
    [root@k8s2 ~]#
    [root@k8s2 ~]# date >> /data/date.log
    [root@k8s2 ~]# date >> /data/date.log
    [root@k8s2 ~]#
    [root@k8s2 ~]# ls /data/
    date.log  dir1  dir2  dir3  dir4  dir5


服务的将客户端加入到配置
==============================

我们先一次性都添加进去，后面再解释内容。  添加下面的内容到文件的尾部。

::

    $ vim /usr/local/bacula/etc/bacula-dir.conf
    FileSet {
      Name = "fileset k8s2 data and etc"
      Include {
        Options {
          signature = MD5
          compression = GZIP
        }
        File = /data
        File = /etc
      }
      Exclude {
        File = /data/dir3
      }
    }

    Client {
      Name = k8s2.alv.pub-fd
      Address = k8s2.alv.pub
      FDPort = 9102
      Catalog = MyCatalog
      Password = "F/FnmGaHpEeoFdSlAfMgmwchYh/eG/tUBd7IFcY1K1EZ"          # password for Remote FileDaemon
      File Retention = 30 days            # 30 days
      Job Retention = 6 months            # six months
      AutoPrune = yes                     # Prune expired Jobs/Files
    }

    Job {
      Name = "job Backup k8s2 data adn etc"
      JobDefs = "DefaultJob"
      Client = k8s2.alv.pub-fd
      Pool = File
      FileSet="fileset k8s2 data and etc"
    }


添加完后我们可以检测一下配置有没有错误，执行下面的命令，如果执行之后没输出任何信息，就是配置语法没有问题，

.. code-block:: bash

    bacula-dir -tc /usr/local/bacula/etc/bacula-dir.conf


下面解释上面的三段配置

添加文件集（服务器）
-------------------------

Bacula FileSet定义了一组文件或目录，用于包含或排除备份选择中的文件，并由Bacula Server上的备份作业使用。

如果您按照设置Bacula Server组件的先决条件教程，您已经有一个名为“Full Set”的FileSet。如果要运行包含备份客户端上几乎所有文件的备份作业，则可以在作业中使用该FileSet。但是，您可能会发现，您通常不希望或不需要对服务器上的所有内容进行备份，并且数据的子集就足够了。

如果文件集中包含的文件更具选择性，这将减少备份服务器运行备份作业所需的磁盘空间和时间。它还可以使恢复更简单，因为您不需要筛选“完整集”来查找要还原的文件。

我们将向您展示如何创建新的FileSet资源，以便您可以更有选择性地备份。

在Bacula Server上，filesets.conf在我们之前创建的Bacula Director配置目录中打开一个名为的文件：

::

    FileSet {
      Name = "fileset k8s2 data and etc"
      Include {
        Options {
          signature = MD5
          compression = GZIP
        }
        File = /data
        File = /etc
      }
      Exclude {
        File = /data/dir3
      }
    }

这个文件中有很多内容，但请记住以下几个细节：
    #. FileSet名称必须是唯一的
    #. 包括要备份的任何文件或分区
    #. 排除您不想备份的，但是由于已经存在于包含的文件中而被选中的文件。
    #. 如果愿意，您可以创建多个FileSet。完成后保存并退出。

现在我们准备创建将使用我们新FileSet的备份作业。

将客户端和备份作业添加到Bacula Server
---------------------------------------------

现在我们准备将我们的客户端添加到Bacula Server。为此，我们必须使用新的客户端和作业资源配置Bacula Director。

添加客户端资源


客户端资源为Director配置连接到客户端主机所需的信息。这包括客户端文件守护程序的名称，地址和密码。

将此客户端资源定义粘贴到文件中。请务必在客户端主机名，私人FQDN和密码（来自客户端bacula-fd.conf）中替换：

::

    Client {
      Name = k8s2.alv.pub-fd
      Address = k8s2.alv.pub
      FDPort = 9102
      Catalog = MyCatalog
      Password = "F/FnmGaHpEeoFdSlAfMgmwchYh/eG/tUBd7IFcY1K1EZ"          # password for Remote FileDaemon
      File Retention = 30 days            # 30 days
      Job Retention = 6 months            # six months
      AutoPrune = yes                     # Prune expired Jobs/Files
    }


您只需为每个客户端执行一次此操作。



创建备份作业
-----------------

备份作业必须具有唯一名称，它定义了应备份哪个客户端和哪些数据的详细信息。

接下来，将此备份作业粘贴到文件中，请注意替换客户端主机名：

::

    Job {
      Name = "job Backup k8s2 data adn etc"
      JobDefs = "DefaultJob"
      Client = k8s2.alv.pub-fd
      Pool = File
      FileSet="fileset k8s2 data and etc"
    }



重启服务
-----------

::

    bacula restart
    bacula status



手动执行备份任务
=====================

::

    bconsole
    run
    (输入我们定义的那个备份任务的编号，比如是4，就输入4)
    yes  （是否允许，输入yes）

    status director  （如果备份的很快，那么稍等片刻后，可以执行status director查看备份状态，看是否完成。如果未完成，报错了，也会显示）



恢复文件
===========

首先，在bconsole里执行restore，


::

    restore
    5  #然后根据我们的实际需求，去选择备份，比如我们要恢复最近的一次备份，那么就选择5
    2 #然后选择客户端，比如这里我们选择第二个客户端，输入2
    ls #然后我们可以执行ls查看本次备份的目录，前面有*的就是当前标记要恢复出来的，我们可以执行mask * 去标记全部，或者unmask 指定文件或目录去取消标记。 如果前面执行的命令是restore all,那么这里会默认标记全部
    mask * #这里我们手动标记全部
    ls
    done #然后输入done，表示标记完毕，开始恢复。

恢复之后，我们也还是可以执行 status director 来查看一下状态。


然后我们可以去客户端k8s2上去确认是否已经恢复，在/tmp/bacula-restore 目录下。

::

    [root@k8s2 ~]# ll /tmp/bacula-restores/
    total 12
    drwxr-xr-x  6 root root   70 Dec 24 21:55 data
    drwxr-xr-x 80 root root 8192 Dec 24 21:39 etc
    [root@k8s2 ~]# ll /tmp/bacula-restores/data/
    total 4
    -rw-r--r-- 1 root root 58 Dec 24 21:56 date.log
    drwxr-xr-x 2 root root  6 Dec 24 21:55 dir1
    drwxr-xr-x 2 root root  6 Dec 24 21:55 dir2
    drwxr-xr-x 2 root root  6 Dec 24 21:55 dir4
    drwxr-xr-x 2 root root  6 Dec 24 21:55 dir5
    [root@k8s2 ~]# cat /tmp/bacula-restores/data/date.log
    Mon Dec 24 21:55:59 CST 2018
    Mon Dec 24 21:56:01 CST 2018

然后可以看到，已经成功恢复，我们在配置定义的不包含dir3,所以dir3没有被备份，所以那样配置也是生效了的。

其他命令
------------
一下命令都是在bconsole终端里执行的

::

    list jobs  #查看任务列表
    llist jobs #查看已执行过的任务详情
    list clients # 查看客户端列表


关于Schedule里时间的定义
================================


::

    = on

    = at

    = 1st | 2nd | 3rd | 4th | 5th | first |

    second | third | fourth | fifth

    = sun | mon | tue | wed | thu | fri | sat |

    sunday | monday | tuesday | wednesday |

    thursday | friday | saturday

    = w00 | w01 | ... w52 | w53

    = jan | feb | mar | apr | may | jun | jul |

    aug | sep | oct | nov | dec | january |

    february | ... | december

    = daily

    = weekly

    = monthly

    = hourly

    = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 0

    = |

    <12hour> = 0 | 1 | 2 | ... 12

    = 0 | 1 | 2 | ... 23

    = 0 | 1 | 2 | ... 59

    = 1 | 2 | ... 31

    = : |

    <12hour>:am |

    <12hour>:pm

    = |


    =

    = -

    = -

    = -

    = | |


    = | |

    = |

    = | |

    | |

    |

    |


    = | |


    =

    针对时间设定规范中的一些词解释一下：

    mday

    月的某一天

    wday

    周的某一天

    wom

    月的某一周

    woy

    年的某一周

    下面举例说明一下

    范例：每天在2:05执行完全备份。

    Schedule {

    Name = "Daily"

    Run = Level=Full daily at 2:05

    }

    范例：周日2:05执行完全备份，周一到周六2:05执行增量备份。

    Schedule {

    Name = "WeeklyCycle"

    Run = Level=Full sun at 2:05

    Run = Level=Incremental mon-sat at 2:05

    }

    范例：每月第一周的周日2:05执行完全备份，每月第二到第五周的周日2:05执行差异备份，周一到周六2:05执行增量备份。

    Schedule {

    Name = "MonthlyCycle"

    Run = Level=Full Pool=Monthly 1st sun at 2:05

    Run = Level=Differential 2nd-5th sun at 2:05

    Run = Level=Incremental Pool=Daily mon-sat at 2:05

    }

    范例：每月1日2:05执行完全备份，其余日期2:05执行增量备份。

    Schedule {

    Name = "First"

    Run = Level=Full on 1 at 2:05

    Run = Level=Incremental on 2-31 at 2:05

    }

    范例：每10分钟执行完全备份。

    Schedule {

    Name = "TenMinutes"

    Run = Level=Full hourly at 0:05

    Run = Level=Full hourly at 0:15

    Run = Level=Full hourly at 0:25

    Run = Level=Full hourly at 0:35

    Run = Level=Full hourly at 0:45

    Run = Level=Full hourly at 0:55

    }


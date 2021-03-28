---
title: Docker配置MYSQL集群(上)
date: 2021-01-27 10:41:35
tags:
    - Docker
    - MYSQL集群
categories:
    - Spring
toc: true
---
利用Docker部署MYSQL集群，可以缓解服务器的压力，接下来一起来学习使用Docker部署MYSQL集群吧！
### <span style="color:orange;">1.MYSQL集群读写分离的原理知识</span>
> MYSQL集群是为了减轻服务器负担，防止网站面对业务过多，达到数据库操作瓶颈，防止用户的数据没有及时的记录或修改造成损失，其中MYSQL集群的主要目的就是设立一个master节点，主要负责写，还有若干个salve节点，负责读，其中涉及到master、slave数据一致性的问题。

<!--more-->

#### <div align=center><img src="集群原理.png" width="50%" height="50%" align=center/></div>

#### 操作步骤

1. 首先master在接收到上层的一个写请求，会将操作记录到一个二进制日志当中。在每个事务更新数据之前完成之前，master会将操作事务串行地写到二进制日志当中，操作事务写入到二进制日志完成之后，master会通知存储引擎提交事务，然后通知slave，这是master上的数据才会进行改动。（这里称对数据的一次操作称之为**一次二进制日志事件**，也就是binary log events)
1. slave会将binary log events拷贝到它的中继日志当中。具体而言，slave会开启一个IO线程，在master上打开一个普通的连接，然后将binary log的变化拷贝到IO线程中，如果binary log没有变化，则会睡眠，并等待master产生一个新的时间，如果有变化，则会把变化部分从内存中拷贝到自己的日志中Replay log中
1. slave重做中继日志事件，将Replay Log中的变化重新在slave做一遍，这里的SQL thread就是从Replay log中读取事件，并重放其中的事件，更新slave的数据，使其与master的数据保持一致

**简单而言：就是master接收到写请求后，会将事务操作日志放入到二进制文件中，然后slave的IO thread会来读取Binary Log，将读取的内容写入到Replay Log，然后SQL Thread再重放事件，更新slave上的数据。**


问：为什么IO thread读取到变化后不经过Replay Log直接重放更新到数据库中？
> 因为计算机角度是需要队列进行的，由于网络原因，IO Thread不可能一次将变化部分全部读取到内存中，此外Slave自身可能还有自己读的业务正在进行，所以Replay Log相当于缓存好master的变化操作，然后进行重放。


### <span style="color:orange;">2. 配置MYSQL集群</span>

经过上面介绍，核心就是Binary Log和Replay Log以及IO thread的配置，首先打开master上的Binary Log位置，然后slave打开自己的Replay Log，然后IO thread连接到master上的Binary Log

#### 1.下载Docker，创建容器
下载好后在终端中输入命令
```bash
docker pull mysql  ###下载mysql
docker images   ###查看下载好的镜像，这里面的tag标签就是下面一行命令中mysql:后面所要填入的版本号
###创建名字为master装载mysql镜像的容器，-d表示后台运行，-p表示端口，10001是宿主端口，3306是所创建容器slave-1的内部端口，其中涉及到内部端口到外部端口的映射，
###其中外部端口号可以用来用Navicat连接master数据库，但是后面配置slave连接master仍然用的是3306容器内部的端口号
docker run --name master -p 10001:3306 -e MYSQL_ROOT_PASSWORD=make1234 -d mysql:latest  
docker run --name slave -p 10002:3306 -e MYSQL_ROOT_PASSWORD=make1234 -d mysql:latest  
docker container ps -a  ###列出所有容器
```
#### <div align=center><img src="镜像.png" width="50%" height="50%" align=center/></div>
<div align=center><img src="navicat.png" width="50%" height="50%" align=center/></div>


#### 2. 首先配置master，开启log-bin日志

```bash
sudo chmod 755 /etc/my.cnf ###给master的配置文件赋予超级管理员权限，否则会出现修改后无法保存已经文件的错误
sudo vim /etc/my.cnf       ###超级管理员权限编辑my.cnf
        添加上 log-bin=master-bin   ###打开master的binary log日志
                    log-bin-index=master-bin.index
          server-id=1
        键入:wq保存退出
 sudo /usr/loca/mysql/support-files/mysql.server restart   ###重启mysql，让配置文件生效
```
修改完成后，进入mysql中，如果显示以下，则表明已经生效
#### <div align=center><img src="master配置信息.png" width="50%" height="50%" align=center/></div>
<div align=center><img src="截屏2021-01-20 12.07.01.png" width="50%" height="50%" align=center/></div>


#### 3. 进入Docker中创建好的slave容器，开启relay-log日志

```bash
docker exec -it slave bash
apt-get update  ###配置国内镜像源
apt-get install -y vim   ###在容器中安装vim
find / -name my.cnf ###查看容器下的my.cnf位置
vim /etc/mysql/my.cnf
        添加上server-id=3  ###这里的id千万不能与master的id重复，如果有其他slave也不能与之重复
    replay-log=slave1-replay-bin
    replay-log-index=slave1-replay-bin.index
    etc + :wq保存退出
exit; ###退出容器
docker restart slave ###重启slave容器，让修改配置生效
docker exec -it slave bash ###进入容器

### 如果这时执行docker exec -it slave-1 bash 会显示container is not running，因为docker此时容器已经退出，无法再继续进行
###迂回方法：如果无法启动主线程，可以使用commit方法将该镜像放入到一个新的容器中，然后执行
###方法在下面第三张图片中
# yizhichangyuan @ yizhichangyuandeMacBook-Pro in ~ [14:32:34]
$ docker exec -it slave bash
Error response from daemon: Container 895994c73c773b9e58ec3b87e6d4e04dad66ae5376f88b1ee134079bd064dcba is not running
# yizhichangyuan @ yizhichangyuandeMacBook-Pro in ~ [14:32:45] C:1
$ docker commit 895994c73c77
sha256:f5baa2c4fff010c9ea90c324e5ae2a174fd6a62b0e2a50e0126be439c4d82293
# yizhichangyuan @ yizhichangyuandeMacBook-Pro in ~ [14:32:57]
$ docker run -it f5baa2c4fff010c9ea90 bash
```
<div align=center><img src="截屏2021-01-20 14.30.05.png" width="50%" height="50%" align=center/></div>
<div align=center><img src="截屏2021-01-20 23.26.12.png" width="50%" height="50%" align=center/></div>



#### 4. master上创建slave节点来进行访问的通道

```bash
docker inspect master --format '{{.NetworkSettings.IPAddress}}' #查看master的ip，这里为172.17.0.2
docker inspect slave --format '{{.NetworkSettings.IPAddress}}' #查看slave的ip，这里为172.17.0.3
docker exec -it master bash  # 进入master容器
mysql -uroot -pmake1234   #进入master的mysql 
# 在master上创建一个名为repl的用户，ip地址为slave的ip{172.17.0.3}，密码验证方式为mysql_native_password，密码为mysql
# 这里相当于在master上开了一个给slave访问的通道，然后slave通过repl这个用户可以访问到master的日志
create user repl@'172.17.0.3' identified with ‘mysql_native_password’ by 'mysql';
# 给ip为172.17.0.3名为repl的用户进行授权，授权访问的范围为*.*，前面一个*表示所有数据库，后面的*表示所有的表，访问权限为salve备份的权限
# 这里的slave有特定含义，表示从节点的含义，不是指的容器slave的名字
# 如果指定只能访问某个数据库{database}，可以改为{database}.*
grant replication slave on *.* to repl@'172.17.0.3';
flush privileges; # 刷新写入权限
show master staus;
```
<div align=center><img src="截屏2021-01-20 23.33.11.png" width="50%" height="50%" align=center/></div>


#### 5.slave节点连接到master所给的通道上
```bash
docker exec -it slave bash  #进入slave容器
mysql -uroot -pmake1234  #进入slave-mysql
stop slave;
# 这里的端口master_port一定得是容器内部的端口，而不是宿主的端口10001，否则会报错，其中master_log_pos从0开始，
#表示将数据库鬼过去所有的改变都扒到数据库slave上，这指定为0在slave宕机恢复很有作用，如果为其他，表示从那个时间点之后的变化恢复到salve上
change master to master_host='172.17.0.2',master_port=3306,master_user='repl',
master_password='mysql',master_log_file='master-bin.000001’, master_log_pos=0;
start salve;
show slave status\G;
```
> 命令说明：
> master_host：Master 的地址，指的是容器的独立 ip, 可以通过 docker inspect 
> --format='{{NetworkSettings.IPAddress}}' 容器名称 | 容器 id 查询容器的 ip
> master_port：Master 的端口号，指的是容器的端口号
> master_user：用于数据同步的用户
> master_password：用于同步的用户的密码
> master_log_file：指定 Slave 从哪个日志文件开始复制数据，即上文中提到的 File 字段的值
> master_log_pos：从哪个 Position 开始读，即上文中提到的 Position 字段的值
> 转自链接：[https://learnku.com/articles/30439](https://learnku.com/articles/30439)
<div align=center><img src="截屏2021-01-20 23.46.48.png" width="50%" height="50%" align=center/></div>


### <span style="color:orange;">3.向Docker容器导入Sql文件</span>
```bash
docker cp path1 container:path2
```
其中path1是要导入容器的Sql文件路径，path2是存放到容器中的位置，container为容器名称，导入到哪个容器中
<div align=center><img src="截屏2021-01-23 21.59.27.png" width="50%" height="50%" align=center/></div>


### <span style="color:orange;">4.报错处理</span>

- 如果show slave status\G中显示为access denied，则
   - 可能是change master指定的master_port是映射宿主的端口造成的（这里为10001），改为端口号为容器内部端口号3306
   - 还要一种原因是，你使用的master是使用的本地的mysql，而不是docker中部署的mysql，因为本地localhost是没有公网ip的，所以本地mysql默认只能通过本地localhost或127.0.0.1链接，不能用IP链接，**也就不能允许其他机器链接本机的mysql**

- show slave status\G中显示的为caching_sha2_password的错误，则为密码验证方式的问题
   1. 首先在master中改变验证方式
```bash
ALTER USER 'repl'@'172.17.0.3' IDENTIFIED WITH mysql_native_password  BY 'mysql';
flush privileges;
show master status; #重新查看master的状态，因为其中的File和Position已经改变
```
<div align=center><img src="截屏2021-01-20 23.58.28.png" width="50%" height="50%" align=center/></div>

   1. 在slave中重新change master，注意其中的master_log_file和master_log_pos已经改变了
   1. 如果上述还不行，则在salve的my.cnf添加如下一句话，同时重新change master（注意其中master_log_file和master_log_pos已经改变）
```bash
default_authentication_plugin=mysql_native_password
```
<div align=center><img src="image.png" width="50%" height="50%" align=center/></div>

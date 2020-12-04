# Redis 

## 安装

* github下载地址：https://github.com/MicrosoftArchive/redis/releases

* 版本：3.2.1 （微软官方构建）
* 有两种安装方式：
  * msi  ： 类似于windows exe文件安装，  
  * zip  ： 下载后直接解压即可使用

* 配置环境变量：将redis安装路径添加到系统变量中  [可选]

* redis默认使用端口为6379 ，

* 启动和关闭

  ```
  //启动redis服务端
  //redis安装目录下 打开cmd
  redis-server.exe redis.windows.conf
  
  //redis作为windows服务启动方式 （无port参数时，默认6379端口，不指定服务名称时，默认为 Redis）
  redis-server.exe --service-install redis.windows-service.conf --service-name 服务名称 --port 端口号 
  
  //启动服务
  redis-server --service-start
  //停止服务
  redis-server --service-stop
  ```

* 测试

  ```
  //在redis安装目录新打开一个cmd窗口 
  redis-cli
  //进入redis ...
  ```

将redis添加到服务中

```
redis-server.exe --service-install redis.windows-service-6380.conf --service-name Redis6380 --port 6380
```

开启服务

```
net start Redis6380
```

## 集群部署

Redis集群，一般为三种:主从模式、Sentinel模式、Cluster模式，其中主从模式和Sentinel模式需要客户端支持。

### 主从模式

主从模式是三种模式中最简单的，在主从复制中，数据库分为两类：主数据库(master)和从数据库(slave)。

其中主从复制有如下特点：

* 主数据库可以进行读写操作，当读写操作导致数据变化时会自动将数据同步给从数据库
* 从数据库一般都是只读的，并且接收主数据库同步过来的数据
* 一个master可以拥有多个slave，但是一个slave只能对应一个master
* slave挂了不影响其他slave的读和master的读和写，重新启动后会将数据从master同步过来
* master挂了以后，不影响slave的读，但redis不再提供写服务，master重启后redis将重新对外提供写服务
* master挂了以后，不会在slave节点中重新选一个master

#### 工作机制

当slave启动后，主动向master发送SYNC命令。master接收到SYNC命令后在后台保存快照（RDB持久化）和缓存保存快照这段时间的命令，然后将保存的快照文件和缓存的命令发送给slave。slave接收到快照文件和命令后加载快照文件和缓存的执行命令。

复制初始化后，master每次接收到的写命令都会同步发送给slave，保证主从数据一致性。

#### 安全设置

当master节点设置密码后：

* 客户端访问master需要密码

* 启动slave需要密码，在配置文件中配置即可

* 客户端访问slave不需要密码

#### 缺点

从上面可以看出，master节点在主从模式中唯一，若master挂掉，则redis无法对外提供写服务。

#### 部署

* 在redis的安装目录新建文件夹6380,6381,6382，并拷贝redis.windows-service.conf和redis-server.exe 文件到新建文件夹中

* 修改配置文件中的端口，其他根据实际情况调整

  ```
  port 6382
  ```

* 添加redis服务(管理员运行)

  ```
  redis-server.exe --service-install redis.windows-service.conf --service-name Redis6382 --port 6382
  ```

* 启动服务

  ```
  net start Redis6382
  ```

* 连接并设置从节点

  * 通过Redis命令设置（临时设置，重启后不生效）

    连接节点，键入下面命令，将该节点设置为6379的从节点

    ![](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123132307639.png)

  * 修改配置文件设置（不修改配置文件永久生效）

    ![image-20201123133806295](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123133806295.png)

* 测试

  切换到主节点，设置key

  ![](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123132531002.png)

  从主节点和子节点获取key

  ![image-20201123132740677](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123132740677.png)

  从节点根据Key值，均可获取到。

  从节点无法写入。

  ![image-20201123132951289](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123132951289.png)

### Sentinel模式

 在主从模式的基础上，在三个文件夹中分别创建一个sentinel.conf文件

```
port 26379 // 当前Sentinel服务运行的端口
sentinel monitor mymaster 127.0.0.1 6379 2  //redis集群的主节点
sentinel down-after-milliseconds mymaster 5000  
sentinel parallel-syncs mymaster 1  
sentinel failover-timeout mymaster 15000  
```

相关参数：

1.  port :当前Sentinel服务运行的端口 
2. sentinel monitor mymaster 127.0.0.1 6379 2:Sentinel去监视一个名为mymaster的主redis实例，这个主实例的IP地址为本机地址127.0.0.1，端口号为6379，而将这个主实例判断为失效至少需要2个 Sentinel进程的同意，只要同意Sentinel的数量不达标，自动failover就不会执行 
3. sentinel down-after-milliseconds mymaster 5000:指定了Sentinel认为Redis实例已经失效所需的毫秒数。当 实例超过该时间没有返回PING，或者直接返回错误，那么Sentinel将这个实例标记为主观下线。只有一个 Sentinel进程将实例标记为主观下线并不一定会引起实例的自动故障迁移：只有在足够数量的Sentinel都将一个实例标记为主观下线之后，实例才会被标记为客观下线，这时自动故障迁移才会执行 
4. sentinel parallel-syncs mymaster 1：指定了在执行故障转移时，最多可以有多少个从Redis实例在同步新的主实例，在从Redis实例较多的情况下这个数字越小，同步的时间越长，完成故障转移所需的时间就越长 
5. sentinel failover-timeout mymaster 15000：如果在该时间（ms）内未能完成failover操作，则认为该failover失败 

创建服务，并设置自动重启 sc create [服务名]

```
 sc create RedisSentinel6382 binpath= "\"D:\soft\Redis\6382\redis-server.exe\" --service-run sentinel.conf --sentinel --loglevel verbose" start= auto
```

开启服务：

```
net start RedisSentinel6382
```

主节点关闭，检查是否能够切换主节点：

当前6380是主节点

![image-20201123154244457](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123154244457.png)

关闭主节点

![image-20201123154330446](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123154330446.png)

连接6379节点，发现6381为主节点

![image-20201123154431994](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123154431994.png)

日志文件中也可以发现，6381被设置成为了主节点

![image-20201123155111369](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123155111369.png)

6381设置key成功

![image-20201123154527874](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123154527874.png)

重启6380，重启后的节点变为了哨兵节点

![image-20201123155309588](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201123155309588.png)

参考文章：

[redis的Sentinel模式（哨兵模式）的windows安装](https://blog.csdn.net/zhanglong_longlong/article/details/78434122)

[Redis高可用之哨兵模式Sentinel配置与启动（五）](https://www.cnblogs.com/guolianyu/p/10249687.html)

### Cluster模式

###### 目标集群配置

* 6个redis服务器，3主3从

###### 基础配置

* redis文件夹下分别新建 6380、6381、6382、6383、6384、6385，分别对应redis集群使用的端口号

* 从redis安装路径下拷贝redis-server.exe、redis.windows-service.conf文件，粘贴至新建的6个文件夹（从安装包里拷贝全新的文件）

* 分别修改 redis.windows-service.conf 文件

  ```
  port 端口号                                    //端口号对应每个文件夹名称
  
  //#cluster-enabled yes 行改为
  cluster-enabled yes                           //是否开启集群 
  
  //#cluster-config-file nodes.conf 行改为
  cluster-config-file nodes-<端口号>.conf        //生成配置文件
  
  //#cluster-node-timeout 15000 行改为
  cluster-node-timeout 15000                    //配置集群超时时间
  
  //appendonly no 行改为
  appendonly yes                                //开启aof数据保存方式
  
  ```

* 按照上面的步骤将新增的redis添加到系统服务 

###### 安装ruby语言环境

* 下载链接 ：https://rubyinstaller.org/downloads/

* 下载后执行安装文件，可选安装都勾选上

###### 安装redis的ruby驱动 gem

* 下载对应redis版本的驱动文件放到redis文件夹下[点击进入下载页面](https://rubygems.org/gems/redis/versions)

###### 启动集群

* 启动6个redis节点

* redis-cluster目录下打开cmd窗口，输入集群构建命令

  ```
  //命令只需输入一次，之后节点关闭，重启就不需要再执行了  replicas = 1 表示构建一主一从
  redis-trib.rb create --replicas 1 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 127.0.0.1:6385
  ```

  ![image-20201127131007436](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201127131007436.png)
  
  构建完成
  
  检查集群节点状态，连接任意节点
  
  ![image-20201127131619599](https://gitee.com/nikoranger/PicBed/raw/master/PicGo/image-20201127131619599.png)

###### 使用

* 使用Redis客户端Redis-cli.exe来查看数据记录

  ```
  //命令 redis-cli –c –h ”地址” –p "端口号";  c 表示集群
  redis-cli -c -h IP地址 -p 端口号
  ```

* 集群相关命令

  ```
  //查看集群的信息
  cluster info
  
  //查看主从关系
  info replication
  
  //查看各个节点分配slot  共16384个slot
  cluster nodes
  ```

* 删除集群

  停掉所有redis服务，保留redis-server.exe、redis.windows-service.conf文件，删除其他文件

参考文档：

[Redis 下载与安装(Windows版) - 苍 - 博客园 (cnblogs.com)](https://www.cnblogs.com/cang12138/p/8880776.html)

[Redis集群（主从、哨兵、分片）](https://blog.csdn.net/weixin_43751710/article/details/104670498)

[Redis集群增加节点和删除节点](https://www.cnblogs.com/hopeofthevillage/p/11535683.html)
---
title: Sentry平台-为Docker Swarm集群添加SSHFS分布式文件存储
date: "2019-03-28T09:00Z"
description: 本文首先介绍了网易云音乐私有化部署的Sentry平台系统架构和当前业务上遇到的分布式存储问题，最后给出搭建SSHFS存储环境解决该问题的实现步骤。
---

### 前言
本文首先介绍了网易云音乐私有化部署的Sentry平台系统架构和当前业务上遇到的分布式存储问题，最后给出搭建SSHFS存储环境解决该问题的实现步骤。

### Sentry架构
![image](https://p1.music.126.net/khgxZ3hwssENM7Y8gJ--1Q==/109951163959632735.png)
当前Sentry平台的部署采用了三台宿主机构成的Docker Swarm集群。Docker容器内运行的WSGI可理解为错误事件的生产者进程, Redis理解为消息队列，Celery worker为错误事件的消费进程。

### 遇到的问题

Sentry平台提供根据SourceMap解析混淆代码的能力, 比如原始收集到的错误如下：
![image](https://p1.music.126.net/c5ZHOyvdcvZnAD1eAuaqcA==/109951163895796415.jpg)
通过SourceMap解析后：
![image](https://p1.music.126.net/lLtnyXr7IJ6Yj_NZXRrKkw==/109951163895799468.png)
为了保证源代码的安全，sentry提供了[Webpack插件](https://docs.sentry.io/platforms/node/sourcemaps/)，可将打包后的js文件及sourceMap文件提前上传至Sentry后台，在后续收到错误上报时直接从文件系统中读取js及map文件。

Sentry提供了一层filestore抽象配置，用于文件的存储。默认配置下，是写本地磁盘，官网是不建议在生产环境使用的。除此之外Sentry还提供了Google Cloud Storage、Amazon S3 Backend的配置，类似于网易的NOS存储服务。

#### 直接写本地磁盘会遇到的问题？
如下图所示意：当进行文件上传时，Nginx会随机负载到一台机器上，如宿主机A。

![image](https://p1.music.126.net/eRzeeTxkCi0R5km0NaCTMA==/109951163894174028.png)


当前端产生错误上报时，请求可能会是由宿主机C上的消费容器进行处理。消费进程尝试从文件系统中读取js及map文件，由于无法读取到，此次解析就会失败，展示的还是混淆后的代码。

![image](https://p1.music.126.net/txq_IEYwOJsWv78B7gRpFw==/109951163894172228.png)

网上有通过NFS来让跨主机的Docker集群进行文件共享的[方案](https://www.jianshu.com/p/0d59bc614baa?utm_source=oschina-app)，示意图如下：
![image](https://p1.music.126.net/YTdUvXsEA4xdTnnS8fu01g==/109951163894176686.png)
搭建前想找PE同事讨论下能否可以协助搭建，还有方案潜在的风险，PE评估后觉得NFS的方案比较老，不太稳定，不建议去使用。

与同事们讨论后可能的解决方案有：
- 方案一: 将三台云主机迁移至 单台物理机上，但是存在单点的问题，并且后续无法扩展。(这可能是最快的方案)。

- 方案二: sentry提供了亚马逊s3及谷歌云存储的配置，可以参考这两个存储的[实现](https://github.com/getsentry/sentry/tree/master/src/sentry/filestore)，扩展一个NOS的实现。

- 方案三: 搭建SSHFS存储方案。

最终评估后，我们采取了SSHFS的解决方案。下文对该方案的环境搭建做介绍。

### SSHFS
#### 首先提供最权威的搭建参考文档：
- [Docker volume driver文档](https://docs.docker.com/storage/volumes/#use-a-volume-driver)
- [docker-volume-sshfs](https://github.com/vieux/docker-volume-sshfs)

#### 搭建后的存储示意图：
![image](https://p1.music.126.net/R5-6tL4aQ6Yl34TwistODw==/109951163959706351.png)

#### 搭建步骤：
1. 三台机器建立ssh互访
    - 申请一个互访的公共账号
    - 本地创建公私钥对, 可由sa协助将公私钥添加至三台机器
    - 验证3台机器可任意进行ssh免密互访

2. 安装插件
每台机器都需要安装
```
$docker plugin install vieux/sshfs sshkey.source=/home/sentry/.ssh/
```
3. 创建数据卷
每台机器都需要安装
```
$docker volume create -d vieux/sshfs -o sshcmd=sentry@hzabj-music-xxxx-machine3::/home/sentry/data 
 -o port=1046 -o uid=999,gid=999 -o allow_other sshvolume
```
其中uid=999,gid=999 为Sentry的Docker镜像内用户id, 需要登录Sentry容器内检查是否一致，若不一致则修改为Sentry容器内的uid.

4. 测试
在machine1上运行
```
$docker run -it --rm  -v sshvolume:/tmp:nocopy sentrybox /bin/bash
$touch testfile
```
以上我们在/tmp文件夹加下创建了testfile，不出意外的话，在machine3的/home/sentry/data文件夹内会同步出现该文件，验证文件共享成功。

5. 申请一块额外的硬盘，将该硬盘作为共享的数据存储盘
例如申请的新硬盘挂载在 /srv/nbs/0/目录下，则
```
$cd /srv/nbs/0/
$mkdir -p ./home/sentry/data
$mount --bind ./home/sentry/data /home/sentry/data/
```
进行此操作后，/home/sentry/data/文件夹下的文件会存入新的硬盘。而/home/sentry/data/目录，为我们在第二步创建数据卷时指定的共享目录。经过以上配置，三台机器挂载sshvolume后存入的文件，都会落入新的硬盘，达到文件共享的目的。

6. 在Docker Swarm集群中使用
需要在docker-compose.yml文件内配置sshfsvolume, 具体使用可参考以下配置:
![image](https://p1.music.126.net/ZJd6j-m2_VAF0mgCy_SKCw==/109951163959774896.png)

需要注意的是若修改docker-compose.yml后，直接deploy失败的话，需要先执行 docker stack rm 将已有service移除
```
$docker stack rm xxx 
```

再执行一次
```
$docker stack deploy 
```
至此为Docker swarm集群添加sshfs文件共享存储环境搭建完成。

### 参考资料
根据官网权威文档进行操作时，遇到了一些坑，解决遇到问题的参考的文献如下：

#### solution
https://github.com/vieux/docker-volume-sshfs

https://docs.docker.com/storage/volumes/#use-a-volume-driver

#### ssh [public_key]

https://blog.csdn.net/li528405176/article/details/82810342

https://blog.csdn.net/rabit87/article/details/79705163


https://blog.csdn.net/zyf2333/article/details/80373502

https://www.linuxquestions.org/questions/linux-software-2/sshfs-how-to-find-out-cause-of-read-connection-reset-by-peer-message-4175614683/

#### chown permission deied [nocopy]

https://github.com/vieux/docker-volume-sshfs/issues/41

https://github.com/docker/docker.github.io/issues/2992

https://docs.docker.com/engine/reference/run/

#### permission [allow_other]
https://unix.stackexchange.com/questions/146544/chown-permission-denied-on-owned-dir

https://ubuntuforums.org/showthread.php?t=1961204

https://unix.stackexchange.com/questions/37168/unable-to-use-o-allow-other-with-sshfs-option-enabled-in-fuse-conf


https://unix.stackexchange.com/questions/222944/mount-with-sshfs-and-write-file-permissions


https://github.com/docker/for-win/issues/497

https://forums.docker.com/t/volume-not-writable-to-non-root-user-container/36103/3

https://ubuntuforums.org/showthread.php?t=2036686


#### compose [syntax]

https://github.com/vieux/docker-volume-sshfs/issues/48

https://github.com/vieux/docker-volume-sshfs/issues/65

https://github.com/getsentry/docker-sentry/blob/master/9.0/Dockerfile
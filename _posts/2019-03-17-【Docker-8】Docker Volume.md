---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Docker
---

https://mp.weixin.qq.com/s/ysAvzhrZ5vh_i2ek8nH_oQ  
https://deepzz.com/post/the-docker-volumes-basic.html  
https://docs.docker.com/v17.09/engine/admin/volumes/  

1. Volumes
2. Bind Mounts
3. tmpfs Mounts

生产环境中使用Docker的过程中,往往需要对数据进行持久化,或者需要在多个容器之间进行数据共享;容器中管理数据主要有两种方式:
数据卷(Data Volumes):容器内数据直接映射到本地主机环境;
数据卷容器(Data Volume Containers):使用特定容器映射到本地主机环境;
一、数据卷
数据卷时一个可供容器使用的特殊目录,它将主机操作系统目录直接映射进容器,类似与Linux中的monut操作.
数据卷可以提供很多有用的特性:
数据卷可以在容器之间共享和重用,容器间传递数据将变得高效方便;
对数据卷内数据的修改会立马生效,无论是容器内操作还是本地操作;
对数据卷的更新不会影响镜像,解藕了应用和数据';
卷会一直存在,直到没有容器使用,可以安全地卸载它.
1、在容器内创建一个数据卷
在用docker run命令的时候,使用-v标记可以在容器内创建一个数据卷.多次使用-v标记可以创建多个数据卷.
下面创建一个 web 容器，并加载一个宿主机目录到容器的 /var/www/html/目录
在宿主机上创建/web/webapp1 目录，并创建一个 index.html 文件，内容如下：

查看镜像,并使用镜像创建容器:

上面的命令加载主机的 /web/webapp1 目录到容器的 /var/www/html 目录。这个功能在进行测试的时候十分方便，比如用户可以放置一些程序到本地目录中，来查看容器是否正常工作。本地目录的路径必须是绝对路径，如果目录不存在 Docker 会自动为你创建它。
/web/webapp1 目录的文件都将会出现在容器内。这对于在主机和容器之间共享文件是非常有帮助的，例如挂载需要编译的源代码。为了保证可移植性（并不是所有的系统的主机目录都是可以用的），挂载主机目录不需要从 Dockerfile 指定。
挂在的目录可以通过使用docker inspect 容器ID


二、数据卷容器
如果需要在多个容器之间共享一些持续更新的数据,最简单的方式是使用数据卷容器;数据卷容器也是一个容器,但是他的目的是专门用来提供数据卷供其他容器挂载.
首先,创建一个数据卷容器dbdata,并在其中创建一个数据卷挂载到/dbdata:

查看/dbdata目录:

然后,可以在其他容器中使用--volumes-from来挂载dbdata容器中的数据卷,例如创建db1和db2两个容器,并从dbdata容器挂载数据卷:

此时,容器db1和db2都挂载同一个数据卷到相同的/dbdata目录.三个容器任何一方在该目录下的写入,其他容器都可以看到.
在dbdata容器中创建一个sunyinpeng文件,到db1容器内查看:

可以多次使用--volumes-from参数来从多个容器挂载多个数据卷,还可以从其他已经挂载了容器卷的容器来挂载数据卷:

如果删除了挂载的容器(包括dbdata、db1和db2),数据卷并不会被自动删除.如果要删除一个数据卷,必须在删除最后一个还挂载着它的容器时显示使用docker rm -v命令来指定同时删除关联的容器.
使用数据卷可以在容器之间自由地升级和移动数据卷.
三、利用数据卷容器迁移数据
可以利用数据卷容器对其中的数据卷进行备份、恢复,以实现数据的迁移.
1、备份
利用下面的命令来备份dbdata数据卷容器内的数据卷:

首先利用ubuntu镜像创建一个容器worker.使用--volumes-from dbdata参数来让worker容器挂载dbdata容器的数据卷(即dbdata数据卷);使用-v $(pwd):/backup参数来挂载本地的当前目录到worker容器的/backup目录.
worker容器启动后,使用了tar cvf /backup/backup.tar /dbdata命令来将/dbdata下内容备份为容器内的/backup/backup.tar,即宿主主机当前目录下的backup.tar.
2、恢复
首先创建一个带有数据卷的容器dbdata2:

然后创建另一个新的容器,挂载dbdata2的容器,并使用untar解压备份文件到所挂载的容器卷中:


# 三. Volume
Volume机制允许将宿主机上指定的目录或文件挂载到容器里面进行读取和修改操作，有2种Volume声明方式，可以把宿主机目录挂载进容器中，两种方式本质是相同的都是把一个宿主机的目录挂载到容器的/test目录。
```
$ docker run -v /test ...
$ docker run -v /home:/test ...
```
第一种方式由于没有显式的声明宿主机目录，那么会默认在宿主机上创建一个临时目录/var/lib/docker/volumes/[VOLUME_ID]/_data然后挂载到/test目录。

容器在创建之后，仅管已经开启了Mount Namespace 如果没有执行pivot_root或chroot，那么容器内能看到整个宿主机文件系统。容器镜像各个层会保存在宿主机/var/lib/docker/aufs/diff目录下，在进程启动之后会被联合挂载到宿主机/var/lib/docker/aufs/mnt目录中。所以我们只需要在容器创建之后，pivot_root或chroot执行之前，把宿主机指定的目录挂载到宿主机/var/lib/docker/aufs/mnt/[读写层ID]/[目录名] 下，那这个Volume就算完成。

在执行这个挂载的时候，由于容器已经创建，也就是Mount Namespace已经开启了。所以这个挂载只在这个容器内可见，在宿主机上是看不到这个挂载点的，因此就保证了容器的隔离性不会被Volume打破。

Volume挂载用到的技术是Linux的`绑定挂载(bind mount)`技术，允许你将一个目录或文件挂载到一个指定的目录下，这时候对挂载点进行的操作只是发生在被挂载
的目录上，原挂载点的内容不受影响。例如把/home挂载到/test目录下，相当于把/test的inode重定向到/home目录下，对/test目录的任何操作实际修改的是/home目录。
![](https://static001.geekbang.org/resource/image/95/c6/95c957b3c2813bb70eb784b8d1daedc6.png)

容器Volume挂载点内的信息并不会被docker commit提交掉，因为docker commit是发生在宿主机空间上，但是由于Mount Namespace Volume挂载只在容器内可见，宿主机并不知道挂载点存在，因此对于宿主机来说这个挂载点目录始终为空。


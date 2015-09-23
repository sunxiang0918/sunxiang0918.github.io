---
title: <翻译>基于内容地址的Docker Registry2.0
date: 2015-09-20 22:18:58
tags:
- 大数据
- Docker
comments: true
---

#基于内容地址的Docker Registry2.0

由于最近在项目中需要使用`Docker`,并且需要持续集成,因此就研究了一下Docker的`Registry`.突然发现`V1`版本的[Registry](https://github.com/docker/docker-registry)已经被官方打为**废弃**了.新的第二个版本的[Registry](https://github.com/docker/distribution)已经开始提供服务.因此就着重的研究了一下这个,无奈现在相关的文档还非常的少.找到一篇日文的文章感觉还不错,因此就翻译了一下.先申明下我日语也就是玩票的性质,肯定有很多不正确的地方,还望海涵.

---
##[Content Addressable DockerイメージとRegistry2.0](http://deeeet.com/writing/2015/04/20/docker-1_6_distribution/)
Docker1.6已经出了.它增加了很多新的功能,比如:容器与镜像的标记(具体更多的信息可以看RancherOS的[“Adding Label Support to Docker 1.6”](http://rancher.com/docker-labels/)),日志记录的多种驱动等等.这次的Release更新中,我最感兴趣的就是Docker的Image开始基于内容进行寻址(Content-addressable) ([#11109](https://github.com/docker/docker/pull/11109))

到目前为止,Docker Registry通过镜像的名字与Tag与Image进行交互(比如:`tcnksm/golang:1.2`).而Tag是由镜像的作者自己定义的.这就不能保证在相同的Tag下的镜像内容是完全相同的(在GIG中由于使用了commit的哈希值,因此标签不会出现这种情况.)

与`Docker1.6`同时发布的`Registry2.0`([docker/distribution](https://github.com/docker/distribution))为镜像使用了一种不重复的ID(`digest`)来确保同一ID值所引用的镜像的内容始终是相同的(immutable image references)

<!--more-->

###尝试
DockerHub已经开始支持`Registry2.0`,因此可以马上使用这个功能.但是,这次是尝试自己搭建一个私有的注册环境来尝试它的新功能(环境是在OSX上使用 boot2docker)

首先需要安装Registry.与V1相同,Docker已经提供了相应的镜像.

```bash
$ docker run -p 5000:5000 registry:2.0
```

准备一个简单的`Dockerfile`文件来构建我们的`tcnksm/test-digest`镜像.

```bash
FROM busybox
```

```bash
$ docker build -t $(boot2docker ip):5000/tcnksm/test-digest:latest .
```

然后通过`images`命令来验证是否正确.使用`--digests`参数会在结果中显示`digest`信息.与Git的不同,`build`的过程是不会生成`digest`的值的.

```bash
$ docker images --digests
REPOSITORY                               TAG                 DIGEST              IMAGE ID            CREATED             VIRTUAL SIZE
192.168.59.103:5000/tcnksm/test-digest   latest              <none>              8c2e06607696        3 days ago          2.433 MB
```

通过`push`向`Registry`进行推送.`push`的过程会生成`digest`值.

```bash
$ docker push $(boot2docker ip):5000/tcnksm/test-digest:latest
...
Digest: sha256:e4c425e28a3cfe41efdfceda7ccce6be4efd6fc775b24d5ae26477c96fb5eaa4
```

为了替换本地生成的不带`digest`的镜像.我们可以在`pull`的时候,使用`NAME@DIGEST`的方式来代替原来`NAME:TAG`的方式.

```bash
docker rmi $(boot2docker ip):5000/tcnksm/test-digest:latest
```

```bash
docker pull $(boot2docker ip):5000/tcnksm/test-digest@sha256:e4c425e28a3cfe41efdfceda7ccce6be4efd6fc775b24d5ae26477c96fb5eaa4
```

这个时候再使用`images`命令来确认一下.这次的镜像信息中就带有`digest`信息了.

```bash
$ docker images --digests
REPOSITORY                               TAG                 DIGEST                                                                    IMAGE ID            CREATED             VIRTUAL SIZE
192.168.59.103:5000/tcnksm/test-digest   <none>              sha256:e4c425e28a3cfe41efdfceda7ccce6be4efd6fc775b24d5ae26477c96fb5eaa4   8c2e06607696        3 days ago          2.433 MB
```

###Dockerfile
`Dockerfile`的`FROM`语句现在也可以使用`digest`来指定镜像的名字.如果发现通过原来的镜像捕捉任何的东西就更新成为新的镜像,那么这样的操作可以被避免(気がついたら元のイメージ更新されていて完成イメージが意図しないものになっていたということが避けられる．)

```bash
FROM 192.168.59.103:5000/tcnksm/test-digest@sha256:e4c425e28a3cfe41efdfceda7ccce6be4efd6fc775b24d5ae26477c96fb5eaa4
```

```bash
$ docker build .
Step 0 : FROM 192.168.59.103:5000/tcnksm/test-digest@sha256:e4c425e28a3cfe41efdfceda7ccce6be4efd6fc775b24d5ae26477c96fb5eaa4
---> 8c2e06607696
Successfully built 8c2e06607696
```

###镜像的更新
可以通过编辑`Dockerfile`文件然后通过`build`命令来构建新的镜像.

```bash
FROM busybox
MAINTAINER tcnksm
```

```bash
$ docker build -t $(boot2docker ip):5000/tcnksm/test-digest:latest .
```

然后执行`Registry`的`push`操作,于是现在的`digest`会被改变.

```bash
$ docker push $(boot2docker ip):5000/tcnksm/test-digest:latest
...
Digest: sha256:4675f7a9d45932e3043058ef032680d76e8aacccda94b74374efe156e2940ee5
```

###机制
简单的说明一下机制.`digest`不是本地生成的,而是通过`push`操作,在`Registry`一方生成的.

当客户端推送镜像到`Registry`时会同时附带`Image Manifest`(也就是签名).`Image Manifest`也就是Docker镜像内容的一些JSON话的定义.在Golang中的结构就如下所示,包含了镜像的名字,FSLayer的信息等等(manifest的具体定义参见[这里](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-1.md))

```go
type ManifestData struct {
    Name          string             `json:"name"`
    Tag           string             `json:"tag"`
    Architecture  string             `json:"architecture"`
    FSLayers      []*FSLayer         `json:"fsLayers"`
    History       []*ManifestHistory `json:"history"`
    SchemaVersion int                `json:"schemaVersion"`
}
```

你可以调用他的API来查看`Manifest`中的内容.

```bash
$ curl $(boot2docker ip):5000/v2/tcnksm/test-digest/manifests/latest
```

因此,`Registry`是通过下面的函数来根据`Manifest`的元数据来生成`digest`的([registry/handlers/images.go](https://github.com/docker/distribution/blob/master/registry/handlers/images.go))

```go
// PutImageManifest validates and stores and image in the registry.
func (imh *imageManifestHandler) PutImageManifest(w http.ResponseWriter, r *http.Request) 
```

在客户端调用的时候通过消息头`Docker-Content-Digest`发送给对方([API doc](https://github.com/docker/distribution/blob/master/docs/spec/api.md#put-manifest))

```bash
202 Accepted
Location: <url>
Content-Length: 0
Docker-Content-Digest: <digest>
```

###Registry不同的话?
如果发送的`Manifest`内容是相同的话,那么推送到不同的Registry其实是会生成相同的`digest`的.`digest`对于`Registry`来说是全局唯一的.

所以通过上面的方式制作的镜像,如果推送到`DockerHub`中的话,是会得到相同的`digest`的.

```bash
$ docker build -t tcnksm/test-digest:latest .
```

```bash
$ docker push tcnksm/test-digest:latest
...
Digest: sha256:e4c425e28a3cfe41efdfceda7ccce6be4efd6fc775b24d5ae26477c96fb5eaa4
```

###Registry2.0

[Faster and Better Image Distribution with Registry 2.0 and Engine 1.6 | Docker Blog](http://blog.docker.com/2015/04/faster-and-better-image-distribution-with-registry-2-0-and-engine-1-6/)

[docker/distribution](https://github.com/docker/distribution)是一个新的Registr的实现.尝试用来解决现在版本的一些API或安全性的问题.现在的V1版本是使用`Pythone`来实现的,而新的`V2`版本则是使用Go语言来实现的.

新的特征有:

* 重新定义了镜像的`Manifest`([Image Manifest Version 2, Schema 1](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-1.md)) - 可以参阅 [#8093](https://github.com/docker/docker/issues/8093),主要是改善了安全方面的问题
* 新的API([Docker Registry HTTP API V2](https://github.com/docker/distribution/blob/master/docs/spec/api.md),[#Detail](https://github.com/docker/distribution/blob/master/docs/spec/api.md#detail)) - 使用`Manifest V2`来改善URI,如果在`Push/Pull`的过程中程序死掉或异常终止,那么当再开启的时候,会继续执行.(详细的实现的话,目前我还没有看到由Go语言接口实现的客户端实现到底是怎么做的).
* 后端的存储插件化([Docker-Registry Storage Driver](https://github.com/docker/distribution/blob/master/docs/storagedrivers.md)) - 在新版本中,可以选择内存,文件系统,S3,Azure Blob Storage来作为镜像的存储.重新定义了一套Go语言的接口,允许你自己实现不同的存储.
* webhook的实现([Notifications](https://github.com/docker/distribution/blob/master/docs/notifications.md)) - `Push/Pull`到某个endpoint的时候,可以发送相应的事件通知.

目前`dist`命令还只有一些基本框架而已([dist](https://github.com/docker/distribution/tree/master/cmd/dist)).这对Docker镜像的pull/push操作只能使用命令行.作为Docker镜像的下载与运行来说,现在不是分离的了,这点有点讨厌.它试图全部都在这里进行解决.

###References

* [Docker Registry 2.0](https://docs.docker.com/registry/overview/)
* [Docker Registry v2 authentication via central service](https://github.com/docker/distribution/blob/master/docs/spec/auth/token.md)
* [Deploying a registry service](https://github.com/docker/distribution/blob/master/docs/deploying.md)
* [kelseyhightower/docker-registry-osx-setup-guide](https://github.com/kelseyhightower/docker-registry-osx-setup-guide)


---
原文链接: [http://deeeet.com/writing/2015/04/20/docker-1_6_distribution/](http://deeeet.com/writing/2015/04/20/docker-1_6_distribution/)
翻译: [翔妖除魔](sunxiang0918.cn)


---
补充一点我自己的看法.

对于V1的问题主要有:

* 性能:
	* 随机ID重复Push  (docker的id是 客户端随机生成的.)
	* metadata,layer,metadata,layer (链表结构,不能随机读取)
	* python实现
* 安全		
	* 随机image id
	* 无法对内容进行校验
	* 无法确定layer来源
* 其他
	* 相同TAG两次下载 镜像可能是不一样的	
	
而V2的好处就在于 ID是更具内容使用SHA256算出来的.

* Digest(V2中的image Id):
	* 内容可校验
	* 算法可插拔
	* cache友好
	* 服务器端计算,加强安全
	* 统一了存储
	
* Manifest:
	* 包含所有layer信息
	* signature强化校验
	* V1兼容
	
* 性能:
	* Go实现
	* 并行pull
	* 减少了auth流程
	
* 其他:
	* Notification机制
	* 后端存储插件化
	* 全新API
	* 更复杂的鉴权方式

* 存在问题:
	* API缺失:   delete, search,list 没有
	* push/pull 速度有待优化 (gzip解压压缩)
	* 镜像格式和V1 **不兼容**
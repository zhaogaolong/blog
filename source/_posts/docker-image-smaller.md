---
title: docker image 的瘦身
date: 2021-04-25
description: 如何瘦身 docker image
tags:
  - docker
  - kubernetes
categories: docker
---

随着容器化的不断普及，镜像作为容器的基础也是受到了指数型增长，之前有一段时间在搞 docker hub，对镜像瘦身有些许心得，分享给大家。

# 现状

随着 ci/cd 的自动化，越来越多的不是那么优雅的镜像构建全部存储到 docker hub 里，一个镜像的 layer 数量从十几层到几十层不等，中间层出现大量垃圾数据，带来很多负面影响和问题。

举个例子

```Dockerfile
FROM centos:centos7.6.1810
RUN yum makecache
RUN yum install vim net-tools dstat htop -y
RUN yum clean all
```

执行构建

```bash
$ docker build -f Dockerfile -t zhaogaolong/centos-tools:v1 .
Sending build context to Docker daemon  2.048kB
Step 1/4 : FROM centos:centos7.6.1810
 ---> f1cb7c7d58b7
Step 2/4 : RUN yum makecache
 ---> Using cache
 ---> a9bebba7b06e
Step 3/4 : RUN yum install vim net-tools dstat htop -y -q
 ---> Running in 162917d77b35
warning: /var/cache/yum/x86_64/7/base/packages/gpm-libs-1.20.7-6.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Public key for gpm-libs-1.20.7-6.el7.x86_64.rpm is not installed
Public key for perl-Pod-Escapes-1.04-299.el7_9.noarch.rpm is not installed
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-6.1810.2.el7.centos.x86_64 (@CentOS)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
install-info: No such file or directory for /usr/share/info/which.info.gz
Removing intermediate container 162917d77b35
 ---> c866064ab1b2
Step 4/4 : RUN yum clean all
 ---> Running in 17aaa7f6e6cb
Loaded plugins: fastestmirror, ovl
Cleaning repos: base extras updates
Cleaning up list of fastest mirrors
Removing intermediate container 17aaa7f6e6cb
 ---> a5181d724237
Successfully built a5181d724237
Successfully tagged zhaogaolong/centos-tools:v2

```

前后对比一下

```bash
# 基础镜像
$ docker image ls centos:centos7.6.1810
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              centos7.6.1810      f1cb7c7d58b7        2 years ago         202MB

➜ docker image  ls zhaogaolong/centos-tools:v1
REPOSITORY                 TAG                 IMAGE ID            CREATED              SIZE
zhaogaolong/centos-tools   v2                  a5181d724237        About a minute ago   633MB


➜ docker image history zhaogaolong/centos-tools:v1
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
a5181d724237        4 minutes ago       /bin/sh -c yum clean all                        23.3MB
c866064ab1b2        4 minutes ago       /bin/sh -c yum install vim net-tools dstat h…   167MB
a9bebba7b06e        7 minutes ago       /bin/sh -c yum makecache                        242MB
f1cb7c7d58b7        2 years ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>           2 years ago         /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B
<missing>           2 years ago         /bin/sh -c #(nop) ADD file:54b004357379717df…   202MB

```

负面影响：

1. 占用存储空间， 不光占用 docker hub 的空间，还占用容器启动的服务器的存储空间，容器启动量大后成指数型增长。
2. 镜像拉去效率下降。比如我们后端使用 vmware 开源的 harbor 来管理 docker image。服务器的 docker pull 镜像的时候首先过去一个 token ，然后用这个 token 拉去镜像，而 token 过期时间只有 300s，如果有过多的垃圾 layer + 恶劣的网络环境，导致镜像拉去效率直线下降。
3. 容器 io 性能下降（待确认）。

# 镜像瘦身三板斧

1. 命令合并
2. 产出转移
3. 合并数据

## 命令合并

```dockerfile
FROM centos:centos7.6.1810
RUN yum makecache && yum install vim net-tools dstat htop -y && yum clean all
```

重新执行 docker build

```bash
$ docker image ls centos:centos7.6.1810
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              centos7.6.1810      f1cb7c7d58b7        2 years ago         202MB
$ docker image ls zhaogaolong/centos-tools:v2
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
zhaogaolong/centos-tools   v2                  54026c4cd698        2 minutes ago       282MB

$ docker image history zhaogaolong/centos-tools:v2
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
54026c4cd698        3 minutes ago       /bin/sh -c yum makecache && yum install vim …   80.1MB
f1cb7c7d58b7        2 years ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>           2 years ago         /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B
<missing>           2 years ago         /bin/sh -c #(nop) ADD file:54b004357379717df…   202MB
```

咦， 竟然只增加了 80m ，比之前构建的镜像 633MB 节省了 351MB。 确实很有效。

有的时候需要应用在做编译的时候，需要先安装好环境，然后执行编译，没法一步完成。

例如下面的例子

> https://github.com/zhaogaolong/latency

```dockerfile
FROM golang:1.15.6-buster as builder
WORKDIR /workspace
ENV GOPROXY https://goproxy.io,direct
COPY . .
RUN go mod vendor -v && CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -a -o server main.go
```

```bash
$ docker build -f Dockerfile -t zhaogaolong/latency:v1 . --no-cache
$ docker image ls golang:1.15.6-buster
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
golang              1.15.6-buster       5f46b413e8f5        3 months ago        839MB

$ docker image ls zhaogaolong/latency:v1
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
zhaogaolong/latency   v1                  141807be4b7a        11 minutes ago      890MB

```

镜像竟然高达 890MB，基础镜像就高达 839MB，我们继续往下分析。

```
$ docker image history zhaogaolong/latency:v1
IMAGE CREATED CREATED BY SIZE COMMENT
141807be4b7a 23 seconds ago /bin/sh -c go mod vendor -v && CGO_ENABLED=0… 44.4MB
dfddf35c7b12 56 seconds ago /bin/sh -c #(nop) COPY dir:a26e918edfe2912b5… 6.55MB
cd7a6e1187ed 56 seconds ago /bin/sh -c #(nop) ENV GOPROXY=https://gopro… 0B
5d5abce40a7a 57 seconds ago /bin/sh -c #(nop) WORKDIR /workspace 0B
5f46b413e8f5 3 months ago /bin/sh -c #(nop) WORKDIR /go 0B
<missing> 3 months ago /bin/sh -c mkdir -p "$GOPATH/src" "$GOPATH/b… 0B
<missing> 3 months ago /bin/sh -c #(nop) ENV PATH=/go/bin:/usr/loc… 0B
<missing> 3 months ago /bin/sh -c #(nop) ENV GOPATH=/go 0B
<missing> 3 months ago /bin/sh -c set -eux; dpkgArch="$(dpkg --pr… 363MB
<missing> 3 months ago /bin/sh -c #(nop) ENV GOLANG_VERSION=1.15.6 0B
<missing> 3 months ago /bin/sh -c #(nop) ENV PATH=/usr/local/go/bi… 0B
<missing> 3 months ago /bin/sh -c apt-get update && apt-get install… 182MB
<missing> 3 months ago /bin/sh -c apt-get update && apt-get install… 146MB
<missing> 3 months ago /bin/sh -c set -ex; if ! command -v gpg > /… 17.5MB
<missing> 3 months ago /bin/sh -c set -eux; apt-get update; apt-g… 16.5MB
<missing> 3 months ago /bin/sh -c #(nop) CMD ["bash"] 0B
<missing> 3 months ago /bin/sh -c #(nop) ADD file:53e587afdbeaee60c… 114MB

```

最上面层 layer 是无法合并的，因为使用了 dockerfile 的不同关键字，一个是 `COPY` 和 `RUN`。

想法：如果可以把二进制包 copy 到另外一个全新的环境中运行，而废弃构建的镜像。确实可以通过 `copy from` 的解决方案解决。

## 产出转移（copy from）

`COPY FROM` 是可以在构建的的过程中从临时镜像把产出（artifact）copy 到另外一个镜像里，我成前一个镜像为 build 镜像，后者为 release 镜像。

docker file 稍微修改一下

```dockerfile
FROM golang:1.15.6-buster as builder
WORKDIR /workspace
ENV GOPROXY https://goproxy.io,direct
COPY . .
RUN go mod vendor -v
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -a -o server main.go

# add release docker image
FROM centos:7.9.2009 as runtime
WORKDIR /
COPY --from=builder /workspace/server /server
ENTRYPOINT ["bash", "-c", "/server"]
```

```bash
$  docker build -f Dockerfile -t zhaogaolong/latency:v2 . --no-cache

$ docker image ls zhaogaolong/latency
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
zhaogaolong/latency   v2                  d0bed0af30d7        7 minutes ago       210MB
zhaogaolong/latency   v1                  141807be4b7a        17 minutes ago      890MB
```

咦，不错，直接从 890MB 降到了 210MB 了，还是比较可观的。

突然有一个天，需求有变，需要在在运行的镜像里安装软件环境，因为我们的 app 有系统调用。 咦，有办法，我们把前两个方法一起不就好咯。

```dockerfile
FROM golang:1.15.6-buster as builder
WORKDIR /workspace
ENV GOPROXY https://goproxy.io,direct
COPY . .
RUN go mod vendor -v
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -a -o server main.go

FROM centos:7.9.2009 as runtime
WORKDIR /
RUN yum makecache && yum install iproute -y -q && yum clean all # install iproute
COPY --from=builder /workspace/server /server
ENTRYPOINT ["bash", "-c", "/server"]
```

开始执行构建

```shell
$ docker build -f Dockerfile -t zhaogaolong/latency:v3 . --no-cache
$ docker image ls zhaogaolong/latency
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
zhaogaolong/latency   v3                  12eb5e19b424        25 seconds ago      238MB
zhaogaolong/latency   v2                  d0bed0af30d7        17 minutes ago      210MB
zhaogaolong/latency   v1                  141807be4b7a        27 minutes ago      890MB
```

v3 版本只增加了安装软件的大小。感觉棒棒哒。

突然有一天来一个偏痴狂，对代码可读性提出了更高的要求，无法容忍 shell 命令 通过 && 相连。

> 《只有偏执狂才能生存》 -- 英特尔的安迪·格鲁夫

## squash merge layer

还有一个神器，可以把整个 docker build 的过程的 layer 全部合并起来。 它就是 squash

install from https://pypi.org/project/docker-squash/

按照偏痴狂的要求，对代码可读性进行了调整

```dockerfile
FROM golang:1.15.6-buster as builder
WORKDIR /workspace
ENV GOPROXY https://goproxy.io,direct
COPY . .
RUN go mod vendor -v
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -a -o server main.go

FROM centos:7.9.2009 as runtime
WORKDIR /
RUN yum makecache
RUN yum install iproute -y -q
RUN yum clean all
COPY --from=builder /workspace/server /server
ENTRYPOINT ["bash", "-c", "/server"]
```

执行 build

```
$ docker build -f Dockerfile -t zhaogaolong/latency:v4 . --no-cache
$ docker build -f Dockerfile -t zhaogaolong/latency:v4-squash . --no-cache --squash
$ docker image ls zhaogaolong/latency |grep v4
zhaogaolong/latency   v4                  d9deb801e6c0        42 seconds ago      591MB
zhaogaolong/latency   v4-squash           7842e95f9b3f        4 minutes ago       238MB
```

发现 squash 的镜像比没有 squash 的镜像小一半，是什么原理呢？

```bash
➜  latency git:(describe) ✗ docker image history zhaogaolong/latency:v4
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
d9deb801e6c0        2 minutes ago       /bin/sh -c #(nop)  ENTRYPOINT ["bash" "-c" "…   0B
659a87a24dac        2 minutes ago       /bin/sh -c #(nop) COPY file:c4a383448ad0c85d…   6.53MB
879e2921709e        2 minutes ago       /bin/sh -c yum clean all                        24.1MB
c825d62d6ade        2 minutes ago       /bin/sh -c yum install iproute -y -q            114MB
e394bcbef3af        2 minutes ago       /bin/sh -c yum makecache                        243MB
19199ba60f3d        3 minutes ago       /bin/sh -c #(nop) WORKDIR /                     0B
8652b9f0cb4c        5 months ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>           5 months ago        /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B
<missing>           5 months ago        /bin/sh -c #(nop) ADD file:b3ebbe8bd304723d4…   204MB
➜  latency git:(describe) ✗ docker image history zhaogaolong/latency:v4-squash
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
7842e95f9b3f        6 minutes ago                                                       34.2MB              merge sha256:1a5f39954399f2fa4929dd1849d324406641e462fb7495d1e1b5b426f929bbde to sha256:8652b9f0cb4c0599575e5a003f5906876e10c1ceb2ab9fe1786712dac14a50cf
<missing>           6 minutes ago       /bin/sh -c #(nop)  ENTRYPOINT ["bash" "-c" "…   0B
<missing>           6 minutes ago       /bin/sh -c #(nop) COPY file:c4a383448ad0c85d…   0B
<missing>           6 minutes ago       /bin/sh -c yum clean all                        0B
<missing>           6 minutes ago       /bin/sh -c yum install iproute -y -q            0B
<missing>           6 minutes ago       /bin/sh -c yum makecache                        0B
<missing>           6 minutes ago       /bin/sh -c #(nop) WORKDIR /                     0B
<missing>           5 months ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>           5 months ago        /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B
<missing>           5 months ago        /bin/sh -c #(nop) ADD file:b3ebbe8bd304723d4…   204MB

```

惊奇的发现，自动会把这次的构建的 layer 全部 merge 成一个 layer。

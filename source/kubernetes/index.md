---
title: 【译】 在 kubernetes Pod 上挂在文件不删除已有的文件
date: 2021-03-21
tags:
- kubernetes
- configmap

---

> 原文 https://medium.com/hackernoon/mount-file-to-kubernetes-pod-without-deleting-the-existing-file-in-the-docker-container-in-the-88b5d11661a6

当我试图将 congfigmap 挂载到 kubernetes pod 的已存在的的目录里

## 概述

几周前开始，我做了一个 [tweetor](https://github.com/bxcodec/tweetor) 的简单程序，在谷歌云平台（GCP）的 kubernetes 部署的一个简单教程。这是我做的 Kube X-mas 项目的一部分。这个项目是用 bahasa indonesia 编写的一系列文章，文章地址：https://medium.com/easyread/christmas-tale-of-sofware-engineer-project-kube-xmas-9167ebca70d2

在这个项目里，我将部署一个 golang 的应用程序在 GCP 的 kubernetes 集群里。我们知道，在应用程序开发时，必须将程序和配置分离，无论是环境变量，还是配置文件（`.json`， `.yaml`， `.toml`）。所以为了灵活，我们可以改变配置在 `staging`、`production` 或 `testing`阶段。

在这个案例里，我将应用程序构建成 `Docker image` ，然后再将它部署到 kubernetes 里，我将使用 kubernetes 注入配置文件，以便应用程序能与注入的配置一起运行。

因此，在 kubernetes 中，我们能将文件注入（挂载）到容器里。在这个例子中，我将应用程序的配置文件挂载到 docker 容器里。为此，我可以使用 kubernetes 的 `Configmap` 和 `Volumes`, 它将允许我将配置文件注入到 docker 容器里。

## 问题

确切的说，我有一个 Go 的应用程序， 将它构建并封装到 Docker image 里，这个应用程序已经编译和构建并保存到了 docker image 里。

这是 `Dockerfile`

```docker
FROM golang:1.11.4-alpine as builder

RUN apk update && apk upgrade && \
    apk --no-cache --update add git make && \
    go get -u github.com/golang/dep/cmd/dep

WORKDIR /go/src/github.com/bxcodec/tweetor

COPY . .

RUN make engine

## Distribution
FROM alpine:latest

RUN apk update && apk upgrade && \
    apk --no-cache --update add ca-certificates tzdata && \
    mkdir /app
### My working directory
WORKDIR /app

EXPOSE 9090

COPY --from=builder /go/src/github.com/bxcodec/tweetor/engine /app

CMD /app/engine
```

因此，它将构建出一个 Docker image, 在该 Docker image 的 `/app/engine` 的位置存放了一个 go 的编译完的可执行程序。但是这个应用程序需要注入配置文件，它才能运行。

![](https://gitee.com/zhaogaolong/img/raw/master/20210322005125.png)

然后，我使用 kubernetes 的 `Configmap` 注入配置文件到应用中

```yaml
apiVersion: v1
data:
  config.toml: |
    title = "Tweet Service Configuration"
    # address
    address = ":9090"
    # redis
    [redis]
        address = "redis.staging:6379"
        db = 0
        pass = ""
    # context
    [context]
        timeout = 2
kind: ConfigMap
metadata:
  name: vol-config-tweetor-api
  namespace: staging

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tweetor-api
  namespace: staging
spec:
  selector:
    matchLabels:
      app: tweetor-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: tweetor-api
      annotations:
        checksum/deployment: TMP_DEP_CHECKSUM
    spec:
      containers:
        - name: tweetor-api
          image: bxcodec/tweetor:latest
          imagePullPolicy: "Always"
          ports:
            - containerPort: 9090
          readinessProbe:
            httpGet:
              path: /ping
              port: 9090
            initialDelaySeconds: 15
            periodSeconds: 15
            timeoutSeconds: 10
          livenessProbe:
            httpGet:
              path: /ping
              port: 9090
            initialDelaySeconds: 15
            periodSeconds: 15
            timeoutSeconds: 10
          resources:
            limits:
              cpu: 100m
              memory: 400M
            requests:
              cpu: 50m
              memory: 200M
          volumeMounts:
            - name: configs
              mountPath: /app
      volumes:
        - name: configs
          configMap:
            name: vol-config-tweetor-api
```

然而，问题发生了。我期望的是将配置文件注入到和应用程序相同的目录，假设工作目录是 `/app`。

之前

```bash
app
└── engine
```

我期望挂载 configmap 后的状态

```bash
app
├── config.toml
└── engine
```

它发生了什么：

```bash
app
└── config.toml
```

kubernetes 挂载卷替换了所有的文件，并删除了已有的文件，而不是添加一个文件。

我在这个问题上花费了 4 个小时，因为我没有意识到这一点。我的 Pod 一直崩溃并重启。在这之前，我尝试 debug 它，并意识到如果在相同的目录下，挂载了一个文件，也会删除现有的文件。

## 解决方案

在意识到这个问题之后，我有两个选择。

- 将 docker 容器的工作目录和我的应用程序的工作目录分开，就是替代使用`/app` 这个工作目录，我将创建一个新的目录包含我的 `WORKDIR`, 将配置文件放到里面
- 弄清楚这一切，并解决掉它

选项 1 能解决我的问题，当时我面临此问题，也是选择了选项 1。 但过去了几天后，我出于好奇，选择了选项 2，我尝试再次阅读 官方文档博客，这个问题发生在别人身上，以及如何解决它。

我什么也没有找到，除了这个文章 https://blog.sebastian-daschner.com/entries/multiple-kubernetes-volumes-directory。这个文件解释了多个 kubernetes 目录挂载，我尝试它是否能解决我的问题。

在这个文章中，它尝试使用 `subPath`, 然后我尝试在 kubernetes 中我的应用的 deployment 使用 `subPath`。

之前

```yml
- name: configs
  mountPath: /app
```

之后

```yml
- name: configs
  mountPath: /app/config.toml
  subPath: config.toml
```

咦，它可以工作了，它不会替代和删除目录中已经存在的文件。

我的 deployment 的 全部 yaml

```yml
apiVersion: v1
data:
  config.toml: |
    title = "Tweet Service Configuration"
    # address
    address = ":9090"
    # redis
    [redis]
        address = "redis.staging:6379"
        db = 0
        pass = ""
    # context
    [context]
        timeout = 2
kind: ConfigMap
metadata:
  name: vol-config-tweetor-api
  namespace: staging

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tweetor-api
  namespace: staging
spec:
  selector:
    matchLabels:
      app: tweetor-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: tweetor-api
      annotations:
        checksum/deployment: TMP_DEP_CHECKSUM
    spec:
      containers:
        - name: tweetor-api
          image: bxcodec/tweetor:latest
          imagePullPolicy: "Always"
          ports:
            - containerPort: 9090
          readinessProbe:
            httpGet:
              path: /ping
              port: 9090
            initialDelaySeconds: 15
            periodSeconds: 15
            timeoutSeconds: 10
          livenessProbe:
            httpGet:
              path: /ping
              port: 9090
            initialDelaySeconds: 15
            periodSeconds: 15
            timeoutSeconds: 10
          resources:
            limits:
              cpu: 100m
              memory: 400M
            requests:
              cpu: 50m
              memory: 200M
          volumeMounts:
            - name: configs
              mountPath: /app/config.toml
              subPath: config.toml
      volumes:
        - name: configs
          configMap:
            name: vol-config-tweetor-api
```

我解决了这个问题的时候，我也学习到了一些重要的东西。当我独自解决这个问题的时候，甚至坚持了 4~6 个小时, 但我在这里学到了宝贵的一课。我期待下一次冒险。

无论如何，你觉得这个有用可以分享这个故事。这样大家就不会掉到同一个坑里，并跟随我继续我的软件工程的学习之旅 :)

引用：

- https://blog.sebastian-daschner.com/entries/multiple-kubernetes-volumes-directory


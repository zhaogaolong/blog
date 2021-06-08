---
title: go mod use ssh
date: 2021-06-08
description: go 下载依赖在 https 不满足的情况下使用 ssh 方式解决。
type: "tags"
tags:
  - golang
  - tips
categories: golang
---

# go mod use ssh

## 概述

最近在做一个权限管理的模块，基于 kubernetes， 使用 kubernetes crd 进行存储，最终渲染成 [authorization-policy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) 实现 mesh 的资源访问控制

代码托管是第三方的 git 代码管理服务，只能通过 http 或 ssh 下载和上传代码。因外部环境和政策的问题（fuck），没法使用协议默认的端口, http 使用 80, ssh 使用 23 端口。在 go get crd 的时候就出现访问 https 的问题，一直无法成功下载依赖，断断续续搞了好几天才搞定。最终小伙伴 [华大猫](https://juejin.cn/user/729731453420792/posts) 找到了解决方案，彻底解决了这个问题。

## 问题分析

1. go get 只支持 ssh 和 https 这种安全的方式下载依赖代码，
2. 默认 go get 时候使用 https 的 query 查找接口，因为没有我们的代码托管服务并没有提供基于 https 查询接口
3. 最终只能全链路使用 ssh 的方式进行 query 和 download

环境介绍：

| 名称                  | 版本               | 备注                   |
| --------------------- | ------------------ | ---------------------- |
| go version            | 1.15               |                        |
| git repo              | projectA, projectB | projectA 依赖 projectB |
| Git server 服务器端口 | 23                 |                        |

## 方案步骤

1.  projectB 的 go.mod 增加 .git 后缀， 下面是 projectB 的 `go.mod`

```bash
module GIT_SERVER/zhaogaolong/projectB.git  // 添加 .git 后缀

go 1.15

require (
	github.com/go-logr/logr v0.2.0
)
```

2. import 时候增加 .git 后缀

```go
	import crdv1 "GIT_SERVER/zhaogaolong/projectB.git"  // projectA import projectB
```

3. ssh config 配置主机默认端口

```bash
$ cat .ssh/config
Host GIT_SERVER
  User zhaogaolong
  Port 23
```

4. .gitconfig 增加 insteadOf 配置，将 https 替换成 ssh

```bash
$ cat projectA/.gitconfig
[url "ssh://zhaogaolong@GIT_SERVER/"]
    insteadOf = https://GIT_SERVER/
```

5. 配置 go env GOPRIVATE

```bash
$ go env -w GOPRIVATE=GIT_SERVER
```

## 总结

go mod 时候会经历几个阶段

1. query 依赖的软件包中的 go.mod 中的 `module` 和 import 名称是否一致
2. Download 依赖包
3. 查询 依赖包的 tag，这里使用 git ls-remote 查询依赖包的 tag 信息
4. 把版本信息写到 go.mod 文件里

问题是解决了，但也留下了一点后遗症，projectB 的代码开发 import 的时候全部需要加上 .git 后缀，很恶心，尝试使用 `go mod replace` 解决，最终是以失败而告终。如果有更优雅的解决方案，[欢迎留言](https://github.com/zhaogaolong/zhaogaolong.github.io/issues)

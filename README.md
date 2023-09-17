# Read Pull Requests

探索开源，从读 Pull Request 开始。

## PR05: [loki/10587](https://github.com/grafana/loki/pull/10587/files)

loki 是一个借鉴了 Prometheus 的设计理念的日志聚合工具，不过采集的不是监控指标而是日志，它是 Grafana 系的工具，满足可观测性需求。

这个 PR 的标题是 "Fix bug in cache of the index object client"，修复一个索引对象客户端的 bug。索引对象客户端是个啥呢？先了解一下什么是对象存储（object storage）。

对象存储是一种适用于非结构化数据的存储方案，相对于文件存储、数据库存储等概念，它将数据包装为对象，并且通过一个唯一标识符（通常是一个键）作为索引来访问，具体可以参考我在另一个 repo 的[整理](https://github.com/kanlac/dailyprompts/blob/main/2023/06/data-storage-schemes.md)。

因此索引对象客户端就可以理解为做对象存储的客户端，它负责：

1. 上传日志数据到服务端
2. 从服务端查询日志数据（会使用缓存）

再看看这个 PR 具体是要修复什么问题呢？贡献者说的是 "missing results when querying logs older than what is kept on ingesters"，就是说当查询较早的日志时会缺少结果，原因是在默认设置下 "the sync is only performed once on startup, but not subsequently"，即缓存的同步只会在客户端创建时执行一次，之后不再执行同步，导致任何新添加到的对象存储的索引都不会加入缓存。

解决方案是改变客户端缓存更新策略，"If a table is not found locally in cache, it performs a lookup against object storage"，如果本地缓存没有找到表，就执行一次对象存储的查找。

了解了具体问题和解决方案后，就可以看代码了。这个 PR 很贴心的提供了单元测试，还提供了丰富的注释，非常易读，所以我这里也就不必要做解释了。值得一提的是单元测试工具 stretchr/testify 中 `assert` 包和 `require` 包的差别：后者是对前者的封装，条件不满足时会直接 `t.FailNow()`，更简单易用。

@*Sep,17*

## PR04: [rclone/7273](https://github.com/rclone/rclone/pull/7273)

本周 PR 的关键词是「发送 systemd 通知」和「代码重构」。

照例还是先介绍一下这个 PR 所在的仓库。当我们想要在不同机器间拷贝或同步文件的时候，我们会用 scp 或者 rsync 这样的工具，而 rclone 就好比是云存储的 rsync，它是一个很成熟的基于 Go 编写的 cli 工具，可以让我们在云存储服务和本地机器之间拷贝或同步文件，支持 Google Drive、阿里云盘等流行云存储服务。

rclone 提供十余种子命令，除了基本的 `ls` `copy` `move` `mkdir` 外，还有 `dedupe` 文件去重、`mount` 挂载远程目录到本地等等进阶功能。真是无愧其「云存储的瑞士军刀」的称号。

在实际体验中，通过几步交互式操作，我很快就创建了一个 Google Drive 连接，可以在终端中轻松地在我的机器和云端互相传输文件，非常方便。

这个 PR 的目的是修复 issue 5117，可以先看一下原 issue 的内容，这是维护者 @ncw 在两年前创建的，他提出由于多个子命令有用到重复的代码，所以可以考虑提取一个公用函数，提高代码复用率。

具体来说，`rclone mount` `rclone serve` 等进阶命令都要借助系统服务完成，在初始化完成后需要通知 systemd 服务已经就绪（ready），优雅退出时则需要发送一个 stop 状态。可以把这些重复的事情提取为一个 Notify 函数，它会返回一个 finalise 函子，供主函数 defer 调用。

进行系统服务通知用到了 [go-systemd](https://github.com/iguanesolutions/go-systemd) 这个库，这个库的作用是告诉 systemd 你的程序启动/停止/重新加载了。

这个 PR 涉及的变更非常简单，而且实际上维护者在 issue 中已经把需求描述的相当清楚了，不过直到两年后才有一个贡献者（@eNV25）留言表示愿意接下这个任务（Can I try it?），维护者很愉快地同意了。贡献者说他笔记本电脑的充电器坏了，买了新的就开始做。不到一周后，贡献者提交了他的变更代码，几乎就是直接用了维护者在需求描述中的代码片段，连注释也没有改。

就这么简单。其实如果维护者自己做的话几分钟就改完了，不过把这种不重要不紧急的任务委派出去，也能让更多新人有开源的参与感，也还是不错的。

@*Sep,10*

## PR03: [go-redis/2675](https://github.com/redis/go-redis/pull/2675/files)

如标题所示，这个 PR 的目标是为 Redis 官方 Go 客户端添加 Gears 支持。

[Gears](https://oss.redis.com/redisgears/) 是 Redis 的一个特性，它是 Redis 的数据处理引擎，支持基于 Redis 数据的事务、批处理、和事件驱动处理，说白了就是可以对 Redis 数据跑一些 Python 脚本。

Gears 并不是 Redis 自带的，而是属于一个扩展模块，如果习惯用 Docker 部署，需要选择包含该扩展的 Redis 镜像（其中包含了 Python 解释器）：

```go
docker run -d --name redisgears -p 6379:6379 redislabs/redisgears
```

执行以上代码启动一个 Redis 实例，我们就可以准备测试使用 Gears 了。这里略过往 Redis 实例中添加数据的操作，直接展示通过 redis-cli 执行 Gears 的方法：

```bash
docker exec -it redisgears redis-cli
127.0.0.1:6379> RG.PYEXECUTE "GearsBuilder().run()"
1) 1) "{'event': None, 'key': 'user:2', 'type': 'hash', 'value': {'age': '25', 'name': 'Bob'}}"
   2) "{'event': None, 'key': 'user:1', 'type': 'hash', 'value': {'age': '35', 'name': 'Alice'}}"
   3) "{'event': None, 'key': 'user:3', 'type': 'hash', 'value': {'age': '40', 'name': 'Charlie'}}"
   4) "{'event': None, 'key': 'user:4', 'type': 'hash', 'value': {'age': '28', 'name': 'Diana'}}"
2) (empty array)
127.0.0.1:6379>
```

这是最简单的一个执行 Gears 函数的方法，在 redis-cli 中输入 `RG.PYEXECUTE`，紧跟着是 Python 代码。这个函数什么都没有做，它只是对现有数据做了一个无操作的批处理。我们可以看到 Gears 函数的返回包含两个数据，第一个是返回的数据，第二个是错误。

建议将编写 Gears 的脚本存储到一个文件，然后通过这样的方式传递给容器内的终端：

```bash
cat mygear.py | docker exec -i redisgears redis-cli -x RG.PYEXECUTE
```

接下来我们尝试做一个简单的数据处理，不过这次我们不用 redis-cli，而是用 go-redis 来执行：

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/redis/go-redis/v9"
)

func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
		DB:   0, // 使用默认 DB
	})

	script := `
def maximum(a, x):
	''' Returns the maximum '''
	a = a if a else 0  # initialize the accumulator
	return max(a, x)

gb = GearsBuilder()
gb.map(lambda x: int(x['value']['age']))
gb.accumulate(maximum)
gb.run('person:*')
`
	val, err := rdb.Do(context.TODO(), "RG.PYEXECUTE", script).Result()

	if err != nil {
		log.Fatalf("Could not execute script: %v", err)
		return
	}

	fmt.Println("Script executed:", val)
}
```
这段 Go 代码演示了 Gears 的基本用法，它会打印 Redis 中所有 "person:" 开头的数据中的最大的 `age`。通过这样的方式编写脚本，甚至引入第三方库，可以让 Redis 完成更为复杂的数据处理操作。

以上就是第三周的 read pull request。本来还想再跑跑测试用例看看，但我根据官方文档执行 `REDIS_PORT=6379 go test` 未能成功，时间原因这里就不再深入了。

@*Sep,4*

## PR02: [external-dns/2917](https://github.com/kubernetes-sigs/external-dns/pull/2917/files)

这周看的仓库是一个配置为 k8s Ingress 和 Service 配置外部 DNS 服务器的工具，简单来说，比如你购买了 Cloudflare 的 DNS 服务，想要把域名解析到 k8s 资源，就可以使用 external-dns，而 external-dns 支持许多主流的 DNS 供应商，包括国内的腾讯云。

为了支持不同的 DNS 供应商，工具需要配合后者的 API 进行修改，而这个 PR 的目的便是兼容 Exoscale API v2，合并者是来自 Exoscale 的开发人员。

这个 PR 历时长达一年，看起来是和 v2 版本的 API 同步进行的。不过相对于 PR01，这一条合的比较顺利。

因为 Exoscale DNS 服务是付费的，这里就不进一步测试了。

@*Aug,24*

## PR01: [TTPForge/239](https://github.com/facebookincubator/TTPForge/pull/239)

在解释这条 PR 之前，先介绍其中提到的几个工具。

首先是 [CodeQL](https://codeql.github.com/)，这是一个代码审计工具，支持扫描分析多种编程语言所编写程序的安全漏洞，同时支持很多流行的开发框架，包括 gorm, logrus 等等。

然后是 [pre-commit](https://pre-commit.com/)，这是一个 Python 编写的工具，可以为 git 操作创建钩子，比如在执行 `git commit` 之前执行一些脚本。很有用的工具，还支持 CI 自动化。更强大的是，还可以引用 GitHub 上其他仓库中的钩子定义！

最后是 [Mage](https://magefile.org/)，这是一个构建工具，和 Makefile 不一样的是，它可以直接把 Go 函数作为 runnable target。通过在 Go 文件中添加以下构建标签，就可以指定只有在执行 mage 命令时才编译这个 go 文件：
```go
//go:build mage
```

介绍完这些工具，我们看看它们是如何整合到 TTPForge 项目中去的呢？

一个是本地执行的 pre-commit 钩子，即由 `.pre-commit-config.yaml` 定义的内容，这个比较好理解。

另一个是 CI，从 `.github/workflows/pre-commit.yaml` 可以看到有执行 `mage installDeps` 任务安装一些公开的 Go pre-commit 库，接着执行 `mage runPreCommit` 任务执行 pre-commit 钩子。

从 Dockerfile 可以看到，pre-commit, mage 和 pre-commit 钩子也做到镜像里面，尝试构建容器的时候也会执行 pre-commit 钩子。

再回过头来看一下这条 PR 做了哪些有趣的改动。

贡献者（@d3sch41n）修复了一些 CodeQL 检测出来的错误，删掉了一些没有使用的死代码。此外，贡献者还提出把 `.pre-commit-config.yaml` 文件中的 Go tests 部分去掉，理由是 CI 本来已经会执行单元测试，同时 CI 也会执行 pre-commit，没必要让单元测试运行两遍，导致 CI 很慢。

审核者（@l50）反驳了一下这个建议，认为可以保留，因为开发者在 commit 的时候就可以提前发现错误。

贡献者表示不赞成，让开发者自己执行 `go test` 就好了。但最后似乎还是妥协了。

@*Aug,17*

# Read Pull Requests

探索开源，从读 Pull Request 开始。

## PR10: [nilaway/109](https://github.com/uber-go/nilaway/pull/109)

### 一

这个 PR 旨在解决一个经典的 Go 编程中常见的错误。阅读以下代码，能看出有什么问题吗？

```go
func analyzer(funcChan chan functionResult, wg *sync.WaitGroup) {
	
	defer func() {
		if r := recover(); r != nil {
			e := fmt.Errorf("INTERNAL PANIC: %s\n%s", r, string(debug.Stack()))
			funcChan <- functionResult{err: e, index: index, funcDecl: funcDecl}
		}
	}()
	
	defer wg.Done()

	// some other code...
}

func run() {
	var wg = sync.WaitGroup
	funcChan := make(chan functionResult)
	for _, file := range Files {
		wg.Add(1)
		go analyzer(funcChan, &wg)
	}
	go func() {
		wg.Wait()
		close(funcChan)
	}
}
```

答案是两个 defer 语句的顺序不对，因为 defer 是 LIFO 顺序的，下面的 defer 会先执行，导致 channel 关闭，出现 "send on closed channel" 错误。

是的，这个 PR 就是修复了这样一个简单的问题（还加上了测试用例），不过若要解释清楚相关的代码在做什么，就没那么简单了，因为涉及到了 golang.org/x/tools/go/analysis 这个静态分析框架的知识，实现原理非常复杂，所以不再深入。

不过这也是我想要讨论的——读别人的代码，要读到什么程度为止？什么时候可以深入细节，什么时候可以粗略跳过？我想关键是搞清楚目标是什么。例如说我现在的目标是在一个小时内完成这篇短文，而写作短文的目的是通过阅读 PR 感受开源社区里别人是怎么协作的。因此，我需要的是看一条 PR 里面的讨论和涉及代码能给我什么启发，而不必关心一个静态分析框架是如何使用的。这一点很重要，不然漫无目的地阅读代码，很可能效率很低，浪费了时间，最后也没啥收获。

### 二

nilway 是一个静态代码分析工具，用于发现可能导致空指针 panic 错误的代码，它基于 analysis 包实现。analysis 允许你编写一些「分析器」（`analysis.Analyzer`），nilaway 中就包含这样一些：用于分析结构体字段的 `structfield.Analyzer`，用于分析闭包函数的 `anonymousfunc.Analyzer`，用于分析函数契约（contract）的 `functioncontracts.Analyzer`……

慢着，函数契约是什么东西？——原来是 nilaway 中提出的一个概念，通过在函数前面写上特定格式的注释，来表示这个函数的行为。比如

```go
// contract(nonnil->nonnil)
```

这样一行就表示该函数在接受到非空参数时，也一定返回非空结果。

也就是说，nilway 能够将特定格式的注释识别为函数契约，并检查函数的实现是否符合契约。

我没有找到函数契约的相关文档，也许是还处在开发测试阶段，只有一个 [PR](https://github.com/uber-go/nilaway/pull/8) 粗略介绍了这个功能。

### 三

测试用例代码中，又看到了熟悉的 `require` 包（前面的 PR05 也出现过），类似 `ErrorContains` 这样的封装好的函数可以方便地断言错误中的内容。

这周的 PR 就到这里。最后留下两个问题：

- 什么时候用 t.Parallel()?
- go.uber.org/goleak 包有什么用？

@*Nov,26*

## PR09: [gitness/751](https://github.com/harness/gitness/pull/751)

gitness 是 Drone 系的一个包含源码管理的 CI 平台，目前是单独开发的，成熟之后会整合到 Drone 项目。如果只是源码的话，我个人感觉用哪个都差不太多，功能性上应该不会有什么明显的优势。不过在 CI 系统上可以做一些比较。

Drone 和 GitLab CI 或者 Jenkins CI 相比有何优势呢？简单来说，一是配置简单，二是丰富灵活的插件系统，三是容器化（容器化有个好处是可以在本地跑 pipeline）。题外话：有兴趣还可以了解一下 Concurse，用户反馈也很不错，学习曲线稍高，但对于某些团队长远来看或许值得一试。

这次看的 issue 是 14 年的一个很古早的 issue，其中涉及的文件仓库里已经没有了，不过我一开始没有发现。为什么会翻到这条？是因为他们每一条提交后面都带了一个可跳转链接的数字，但实际上都是跳转到无关的 PR 上去了。我猜测这个数字可能是关联社区内部的任务管理系统之类的东西，但是 GitHub 把它们识别为 PR 号了。不过将错就错吧，影响不大。

这个 issue 提了一个很大的变更，不过需求描述的很清楚，目的是简化构建的运行步骤。作者希望最终通过一条 `docker run` 命令就可以完成一个 pipeline，非常简洁优雅。

而且作者建议把这个大的变更拆分成多个 PR，这样一来就能够独立测试，并且能把对系统的影响减到最小，当然 code review 的压力也会更小。这个想法真是不错。

那么这次的 PR 就是为了解决这个 issue 中的第一个子任务——将 SSH 密钥的设置从在容器中完成前置为在宿主机完成。

这时候当然就要问一个问题：这个 SSH 密钥是做什么用的？虽然 issue 和 PR 都没有说明，不过回想一下过去 GitLab 的使用经验，可以合理推测是用于访问源码仓库。通过翻阅源码，可以确认这一点。而且经过几年的迭代，我还发现这个功能如今已经独立到 plugin 仓库维护了，并且就是 GitLab plugin。看来 gitness 在组件化这一块做的是不错的。

回过头来再看一下 issue 中那条 `docker run` 命令：

```bash
`docker run {image} /bin/bash -c "echo {script} | base64 --decode | tee run.sh | sh run.sh && rm run.sh"`
```

有两个有趣的问题值得一提：

1. 为什么在将脚本作为命令的参数传入时，要用 base64 编码？
2. `tee` 命令起什么作用？

我这里就不做解答了，ChatGPT 讲得比我清楚。

最后再看看 PR 讨论记录。贡献者发起 PR 之后，审核者很谦逊地提了两个要求。第一个是关于函数通用性的，`func WriteFile(file []byte, int uint32)` 比 `func WriteKey(key []byte)` 的通用性更好；第二个是关于 Go 占位符的使用，我们知道 `fmt.Printf("%s", v)` 可以以字符串打印 v 变量，不过 `%s` 会进行转义，如果需要保留转义符，那么就需要使用 `%q`，在处理 JSON 的时候需要注意这一点。

@*Nov,12*

## PR08: [zerolog/560](https://github.com/rs/zerolog/pull/560)

Go 1.21 将结构化日志库 slog 包含进了标准库，这周看的是另一个日志库 zerolog，它们其实都是基于 zap 的改良，有着更好的性能，更加轻量，且简单易用。

还有一个 logrus 是我在公司用的，顺便提及一下它和 zap 的区别——Reddit 上有个[帖子](https://www.reddit.com/r/golang/comments/zyfeqm/why_i_chose_zerolog_over_logrus_for_go_logging/)谈到了 logrus 使用过程中遇到的弊端：

1. 比较难配置出想要的输出
2. 高并发场景下内存占用有些高

相对来说，zerolog 更轻量，更易用，更快。而且 logrus 也没有再更新了。

好，进入正题。这次的 PR 标题翻译过来是「在使用 `.Fields` 时显示错误栈」	。看起来就是一个很简单的需求，不过需要知道 zerolog 的基础用法：

```go
log.Error().Err(err).Msg("print some error")
log.Error().Stack().Err(err).Msg("print error trace")
log.Info().Fields(m).Msg("print a map or slice")
log.Info().Stack().Fields(m).Msg("trying to print a map or slice with error trace")
```

确实不能更简单易用，也不用创建或维护什么实例，一行代码就可以输出一条 JSON 日志。`Err` 可以接收一个错误，相应的 `Int` 可以接受整型，非基础数据类型则可以用 `Fields`，它接收 `map[string]interface{}` 或者 `[]interface{}` 两种类型。贡献者提出在使用 `Fields` 的时候不能将其中错误的错误栈打印出来。

拉取最新的代码测试，咦？为什么还是不能把切片中的错误的调用栈打印出来？——原来这个变更在合并到主分支后又被抹除了，连单元测试都找不到了。或许维护者后来认为这个功能意义不大吧……

@*Oct,29*

## PR07: [goreleaser/4324](https://github.com/goreleaser/goreleaser/pull/4324)

[goreleaser](https://goreleaser.com) 是一个发布自动化工具，它的功能包括但不限于：

- 自动推送镜像到 registry
- 自动推送 HomeBrew 包到 tap 仓库
- 自动发布 Winget 包
- ……

goreleaser 可以以容器化的形式整合到你的 CI 环境，也可以直接以 cli 的方式使用。

该 PR 旨在解决 discussion 4321: *How to make a cross-repository pull requests (for Winget) using a private key?* ——如何使用私钥为 Winget 创建一个跨仓库的拉取请求。

讨论的创建者是来自一家芬兰的数字化公司（不错的地方），他表示使用 goreleaser 配置 HomeBrew 包的发布一切正常，发布 Winget 包时却出现了问题。读第一遍的时候我以为是密钥相关的问题，绕了一些弯子，再读两遍，原来是创建提交的过程出了问题。

这里需要了解一下 Winget 包的发布方式，它是需要你往 microsoft/winget-pkgs 这个仓库创建拉取请求，拉取请求被合并，你的包就算发布成功了。

问题描述的片段摘录：

> With Homebrew tap, I can just commit directly to the main branch of the listed repo. With microsoft/winget-pkgs repo I should be **committing to a named branch** in our own futurice/winget-pkgs fork, and then making a cross-repository pull requests to the Microsoft repo.

这里的关键是需要在指定 fork 仓库的指定分支上创建提交，issuer 尝试了两种不同的定义分支的方式，最后都没有效果。

为什么会出现这样的问题呢？goreleaser 的维护者很快就做出了答复，说这是一个 bug，并且已经解决，非常高效。从代码中我们可以看到，的确在原先的代码中没有处理指定分支的情况，而是都默认向主分支提交了。因此 PR 的标题是 *git client should respect specified branch*。

`gitClient` 实现了一个 `CreateFiles` 方法，通过调用方法来在特定 repo 的特定分支下创建 commit，很简单直观，代码可读性很高。

感悟：如果对一个项目不熟悉，光看 issue 或 discussion 描述，还是比较难定位到问题的，不过像这个 PR，一旦能定位到问题就很容易解决了。所以不必指望自己能够轻轻松松为自己并不熟悉的项目解决问题，最好的方式仍然是相关代码的维护者去解决。不过有些时候一些维护者会告诉你解决思路，把机会留给新人，就像我前面的 PR04 那样。

@*Sep,29*

## PR06: [cluster-api/9460](https://github.com/kubernetes-sigs/cluster-api/pull/9460)

cluster-api 是一个用于实现 Kubernetes 平台自动化管理和运维的工具，支持来自多种供应商的集群。CLI 程序提供集群管理的命令，比如 `clusterctl get` 获取集群的负载信息，`clusterctl delete` 从管理集群列表删除等。

这个 PR 旨在完成 issue 9011 相关工作的一部分，这是红帽 OpenShift 公司首席软件工程师发起的重量级 issue，我们看看它的标题：*Importing API types pulls in lots of dependencies*，导入 API 类型时拉取了很多依赖。

问题描述是这样说的：在 Go 代码中导入 cluster-api 的包（`sigs.k8s.io/cluster-api/api/v1beta1`）并执行 go mod tidy，会下载 56 个依赖项，其中还包括 `controller-runtime`。这是没有必要的，比如你只是需要编写一个消费者，那么你并都用不到 CAPI 的代码。

因此，issuer 建议精简依赖项，尤其是把用到了 `controller-runtime` 的代码迁移出去：

> If we modify the way the `SchemeBuilder` works and move the webhooks to a separate package, then we can reduce the required imports for the API package down to just 31 dependencies, and, these dependencies do not include `controller-runtime` or other fast moving/changing packages.

这是一项规模庞大的改动，但仍然获得了评审者的一致通过。在 issuer 描述的下方总共列举了二十余项需要完成的工作——这次的 PR 便是其中之一——目前除了文档部分均已完成。

查看该 PR 的代码变更，可以看到其主要内容是把 webhook 移到了单独的包，显然 webhook 是服务端才会需要的功能。

@*Sep,24*

## PR05: [loki/10587](https://github.com/grafana/loki/pull/10587)

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

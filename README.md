# Read Pull Requests

探索开源，从读 Pull Request 开始。

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

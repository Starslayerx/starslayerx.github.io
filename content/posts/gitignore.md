+++
date = '2025-08-08T8:00:00+08:00'
draft = false
title = 'Git Whitelist'
+++

有时你开启了一个新的项目, 运行了 `cargo init`、`uv init` 和 `go mod init`

这些命令创建了工作所需要的必要文件, 同时也在 `.gitignore` 文件中添加了以下内容
```
target
__pycache__
bin
```
一切都很顺利, 你继续开发新功能, 等到时机成熟时就将项目发布到了 Git 托管平台上

人们开始对你的项目感兴趣, 甚至有人决定为你实现一个新功能, 这简直是免费劳动力!

当你查看代码, 发现了一个格格不入的文件 `.DS_Store`, 你问那个人这是什么, 他说他根本不知道

然后你只是将该文件从分支里面删除, 并把文件名加入了仓库的 `.gitignore`

```
target
__pycache__
bin
.DS_Store
```
现在代码合并到了 `main`, 仓库里只包含有用的内容

接着, 另一人使用基于 Web 技术的 IDE 提交了另一个合并请求, 一看发现有一个完全无关的目录也被提交了, 于是 `.gitignore` 里又增加了一条内容
```
target
__pycache__
bin
.DS_Store
.vscode
```
接下来, 有人使用 IntelliJ IDEA 提交了五百个 XML 文件和 .idea 目录, 这时又不得不将其加入 `.gitignore`
```
target
__pycache__
bin
.DS_Store
.vscode
.idea
```
多年后, `.gitignore` 已经有了上百行, 但是仍然时不时有各种奇怪的文件, 例如 `testscripts`、`foo`、`a`、`qux`、`data.tar.gz`、`start.sh`、`cat` ......

你就像西西弗斯一样, 因欺骗死亡和冥界而受到永无止境的惩罚

> 西西弗斯推着一块写着 `.DS_Store` 的巨石艰难上山

如何改变偷偷溜进来的文件循环呢? 去教育每一个提交合并请求的人肯定不行, 得通过自动化工具解决, 而不是主观沟通

幸运的是, 可以将这个黑名单变成白名单, 可以通过默认忽略所有文件, 然后只手动“取消忽略”明确允许的文件
```
*

!.gitignore

# 白名单：任意位置下的 src 目录及其子文件夹
!src/
!src/**/
!src/**/*.rs
!Cargo.{toml,lock}

# 白名单：项目根目录下的 pysrc 目录
!/pysrc/
!/pysrc/*.py
!pyproject.toml
!uv.lock

!/cmd/
!/cmd/*.go
!main.go
!go.{mod,sum}

!/docs/
!/docs/*.md
```
现在, 没人再能不小心提交不该提交的文件了. Git 会自动忽略所有文件, 只允许那些明确列入白名单的文件.

这种做法也具备一定的“面向未来”的能力——当然, 前提是以后不会有某个 IDE 把 `src/ide.rs` 当成保存项目配置的理想文件路径, 但愿那一天永远不会到来...

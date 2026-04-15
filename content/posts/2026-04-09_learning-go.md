+++
date = '2026-03-02T8:00:00+01:00'
draft = true
title = 'Learning Go 1: Setting Up Environment'
categories = ['Note']
tags = ['Golang']
+++

首先安装 go 语言，其二进制程序位置在 `/usr/local/bin/go`

在 `.zshrc` 中添加以下环境变量：

```bash
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
```

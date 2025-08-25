+++
date = '2025-08-22T8:00:00+08:00'
draft = false
title = 'Rust Alternaitves Tools'
tags = ['Tools', 'Rust']
+++

常用工具的 rust 替代品.

### Introduction
在 Unix 生态中, 许多命令行工具都是用 C 编写的, 经过几十年的优化, 性能和稳定性都非常优秀.
然而, 近年来, Rust 以其安全性、内存管理优势和现代化开发体验, 成为系统级工具开发的理想选择.

首先更新 cargo, 不同系统都可以使用 cargo 安装, 当然也可以使用系统的包管理器安装
```
rustup update stable 
```

有需要的话修改源, 一般在 `~/.cargo/config.toml`, 下面是科大源
```toml
[source.crates-io]
replace-with = 'ustc'

[source.ustc]
registry = "sparse+https://mirrors.ustc.edu.cn/crates.io-index/"

[registries.ustc]
index = "sparse+https://mirrors.ustc.edu.cn/crates.io-index/"
```

### Filesystem & Archiving 文件系统与归档
- [exa](https://github.com/ogham/exa) 替代 `ls`: 彩色支持 Git 状态的 ls 替代品  
    常用参数:
    - `-1`: 一行显示一个文件
    - `-l`: 显示文件细节信息
    - `-F`: 在目录文件名末添加斜杠符号
    - `-T`: 树状显示
    - `-R`: 递归显示所有文件
    - `--icons`: 显示图标
- [zoxide](https://github.com/ajeetdsouza/zoxide) 替代 `cd`: 基于访问频率的快速目录跳转工具  
    常用参数:
    - z foo # 匹配 foo 的路径
    - z foo bar # 匹配 foo & bar 的路径
    - z - # 回到之前目录
    - zi foo # fzf

### File & Text Processing 文件与文本处理
- [bat](https://github.com/sharkdp/bat?tab=readme-ov-file) 替代 `cat`: 具备语法高亮、行号显示、Git 集成等功能, 让查看文件内容更加美观
- [ripgrep](https://github.com/BurntSushi/ripgrep) 替代 `grep`: 使用 Rust 编写的极速文本搜索工具, 支持递归搜索、正则表达式、忽略规则(.gitignore)等
- [fd](https://github.com/sharkdp/fd) 替代 `find`: 提供简单直观的语法、更快的搜索性能, 并默认支持彩色输出和忽略 .gitignore 文件

### System Monitoring & Management 系统监控与管理
- [bottom](https://github.com/ClementTsang/bottom) 替代 `top` / `htop`: 一个现代化的系统资源监控工具, 支持 CPU、内存、磁盘、网络等多种指标显示, 并提供交互式界面
- [procs](https://github.com/dalance/procs) 替代 `ps`: 更人性化的进程信息显示, 支持彩色输出、树状显示、搜索与过滤

## Wrapping up
Rust 的安全性和高性能使其成为编写现代 Linux 工具的理想选择.
这些替代品不仅提供了更好的用户体验, 还利用 Rust 的并发优势和零成本抽象提升了性能.
其他一些 rust 工具:
- uv: python 环境管理工具
- alacritty: 支持 gpu 加速的终端, 实时刷新

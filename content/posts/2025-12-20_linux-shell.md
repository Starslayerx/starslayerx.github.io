+++
date = '2025-12-18T8:00:00+08:00'
draft = false
title = 'Shell'
tags = ['Linux']
+++

## Terminal

终端是提供文本用户界面的程序，早期终端是集成设备，键盘和屏幕集成在一起，现在终端知识简单的应用程序。
除了基本的输入输出外，终端还支持将所谓转义序列或转移码，用于光标和屏幕处理，并可能支持颜色。
例如，使用 `Ctrl-h` 会删除前面一个字符。

环境变量 TERM 可能使用了终端模拟器，其配置可以通过 `infocmp` 获得

```text
#       Reconstructed via infocmp from file: /usr/share/terminfo/74/tmux-256color
tmux-256color|tmux with 256 colors,
        am, hs, km, mir, msgr, xenl,
        colors#256, cols#80, it#8, lines#24, pairs#32767,
        acsc=++\,\,--..00``aaffgghhiijjkkllmmnnooppqqrrssttuuvvwwxxyyzz{{||}}~~,
        bel=^G, blink=\E[5m, bold=\E[1m, cbt=\E[Z, civis=\E[?25l,
        clear=\E[H\E[J, cnorm=\E[34h\E[?25h, cr=^M,
        ...
```

显示内容看上去很混乱，这些是

## Shell

shell 是一个运行在终端内部的程序，充当命令解释器。
shell 通过流提供输入和输出处理，支持变量，有一些内置命令，处理命令执行和状态，支持交互式和脚本使用。
最初的 shell 叫做 Bourne shell sh，即作者名字命名，现在通常被 bash 替代，即 "Bourne Again Shell" 的缩写。

```bash
file -h /bin/sh
```

### Stream

shell 为每个进程提供了三个默认的文件描述符 (FD)，用于输入和输出：

- stdin (FD 0)
- stdout (FD 1)
- stderr (FD 2)

默认情况下，这些 FD 分别连接到屏幕和键盘，即默认从键盘获取输入 stdin，并将输出 stdout 传递到屏幕。
如果不希望屏幕输出 stderr，则可以使用 `$FD>` 或 `<$FD`重定向流。
例如，`2>` 表示重定向 stderr，`1>` 和 `>` 是相同的，都是 stdout，如果想两个都重定向，使用 `&>`。
当想摆脱一个流时，使用 `/dev/null`。

```bash
# 丢弃所有输出
curl https:example.com &> /dev/null

# 将 curl 的标准输出（网页内容）保存到文件
# 将 curl 的标准错误（进度信息）保存到另一个文件
curl https:example.com > content.txt 2> curl_status.txt

# 创建交互式输入文件
cat > temp.txt

# 大写替换为小写
tr < curl_status [A-Z] [a-z]
```

其中 `tr` 命令用于文本转换、删除和压缩，专门处理单个字符串，而不是整行文本。

使用 `<` 重定向的一般是文件，如果是一条命令或者字符串，使用 Here String，比如下面这样

```bash
cat <<< "Hello World"
nvim <<<$(curl https://example.com)
```

而 `<()` 的来源的进程输出，例如比较两个目录内容

```bash
diff <(ls dir1) <(ls dir2)
```

shell 通常会理解一些特殊字符：

- 与字符 `&`

  置于命令的末尾，在后台执行命令

- 反斜杠 `\`

  用于继续下一行的命令，提高长命令的可读性

- 管道连接符 `|`

  连接一个进程的 stdout 与下一个进程的 stdin，允许传递数据，而不必将其存储在文件中

例如下面命令统计收到的网页数据有多少行

```bash
curl https://example.com 2> /dev/null | wc -l
```

其中 `wc` 是一个常用的文本统计命令，全称 word count，下面是一些常用参数：

- `-l` 统计行数
- `-w` 统计单词数
- `-c` 统计字节数
- `-m` 统计字符数
- `-L` 统计最长行的长度

### Variable

在 shell 中经常遇到的术语是变量，当不能蝇编码时，可以使用变量来存储和更改值，可以使用变量来存储和更改值。

要区分两种变量：

- shell 变量

  作用域为当前 shell 会话，不会传递给子进程，使用 `set` 设置 shell 变量或显示环境变量

- 环境变量

  作用域为 shell 及其所有子进程，会自动传递给子进程。

  在 bash 中可以使用 `export` 创建环境变量，`env` 查看环境变量，当想访问一个变量时在它前面放一个 `$` 符号，当想摆脱它时，使用 `unset`

```bash
# 设置 shell 变量
set MY_VAR=42
# 查看 shell 变量
set | grep MY_VAR

# 设置环境变量
export MY_GLOBAL_VAR="fun with vars"

# 查看所有变量
set | grep "MY_*"

# 查看环境变量
env | grep "MY_*"
```

下面是一些常见的 shell 和环境变量

| 变量     | 变量类型   | 语义                                                     |
| -------- | ---------- | -------------------------------------------------------- |
| EDITOR   | 环境变量   | 默认情况下用于编辑文件的程序路径                         |
| HOME     | POSIX      | 当前用户的主目录路径                                     |
| HOSTNAME | bash shell | 当前主机的名称                                           |
| IFS      | POSIX      | 用于分隔字段的字符列表，当 shell 在展开拆分单词时使用    |
| PATH     | POSIX      | shell 在其中查找可执行程序（二进制文件或脚本）的目录列表 |
| PS1      | 环境变量   | 正在使用的主要提示符字符串                               |
| PMD      | 环境变量   | 工作目录的完整路径（应为 `PWD`）                         |
| OLDPWD   | bash shell | 最后一个 cd 命令之前的目录的完整路径（应为 `OLDPWD`）    |
| RANDOM   | bash shell | 0～32 767 之间的随机整数                                 |
| SHELL    | 环境变量   | 包含当前使用的 shell                                     |
| TERM     | 环境变量   | 所使用的终端模拟器                                       |
| UID      | 环境变量   | 当前用户唯一 ID（整数值）                                |
| USER     | 环境变量   | 当前用户名                                               |
| -        | bash shell | 前一个命令在前台执行的最后一个参数                       |
| ?        | bash shell | 退出状态，参见下文“退出状态”                             |
| $        | bash shell | 当前进程的 ID（整数值）                                  |
| @        | bash shell | 当前进程的名称                                           |

shell 使用所谓退出状态命令执行的完成传递给调用者，通常 linux 命令在终止时会返回一个状态。
这可以是一个正常终止，也可以是异常终止。
0 表示命令成功运行，没有错误，1~255 之间非零表示失败，可以使用 `echo $?` 命令查询退出状态。

### 内置命令

shell 有许多内置命令，例如 yes, echo, cat 或 read，有些命令不是内置的，位于 `/usr/bin`。
可以使用 help 命令列出内置程序，使用下面命令找到可执行文件地址

```bash
which ls  # /usr/bin/ls
type ls   # ls is aliased to `ls --color=auto'
```

### Job Control

很多 shell 支持一个特性叫[作业控制](https://www.digitalocean.com/community/tutorials/how-to-use-bash-s-job-control-to-manage-foreground-and-background-processes)。
默认情况下，输入一个命令时，它会控制屏幕和键盘，通常称为前台运行。
但如果不想交互式运行某些东西，可以在命令后面加上 `&` 将前台进程发送到后台，也可以按 `Ctrl-z`。

```bash
# 每 5 秒执行一次 ls，并放到后台
watch -n 5 "ls" &

# 列出所有作业
jobs

# 将后台任务放到前台
fg
```

---

下面是一些常用 shell 快捷键：

- `Ctrl-_`: 撤消
- `Ctrl-r`: 搜索历史记录
- `Ctrl-g`: 取消搜索

这些快捷键都是基于 Emacs 的，可以使用 `set -o vi` 将其变为 vi 模式（感觉命令行下还是 emacs 模式好用）。

### 文件内容管理

```bash
# 覆盖重定向
echo "First line" > file.txt
cat file.txt

# 追加重定向
echo "Second line" >> file.txt
cat file.txt

# 将第一行的小写替换为大写
sed 's/line/LINE/' file.txt
# 全部替换
sed 's/line/LINE/g' file.txt

# Here document
cat << 'EOF' > otherfile.txt
Fisrt line
Second line
Third line
EOF

# 对比两个文件差异
diff -y file.txt otherfile.txt
```

`<< 'EOF'` 是 Here Document 的开始标记，`> otherfile.txt` 是将输出重定向到文件，中间三行是文档内容，最后的 `EOF` 是结束标记。

### 查看长文件

对于长文件，使用 less 或 tail。

```bash
# 循环向文件 tempfile 写入行
for i in {1..100}; do echo $i >> tempfile; done

# 查看前 5 行
head -5 tempfile
```

对于一个不断增长的文件的实时更新，可以使用：

```bash
sudo tail -f /var/log/mail.log
```

### 日期和处理时间

date 命令是生成唯一文件名的有用方法，能够生成各种格式的日期，包括 UNIX 时间戳，以及在不同的日期和时间格式之间进行转换。

```bash
date +%s  # 1766296727
date -d @1766296727 '+%m/%d/%Y:%H:%M:%S'  # 12/21/2025:05:58:47
```

UNIX 是自 1970-01-01T00:00:00Z 以来经过的秒数，UNIX 时间将每天精确地视为 86400 秒。
如果将 UNIX 时间存储在有符号 32 位整数中，那需要注意，这回导致在 2038-01-19 出现问题，计数器将会溢出。

### Zsh

Z-shell 或 zsh 是一个类 Bourne shell，具有强大的补全系统和丰富的主题支持。
zsh 使用 5 个启动文件，如下：

```bash
$ZDOTDIR/.zshenv   # 每次启动时加载，设置环境变量
$ZDOTDIR/.zprofile # 登陆 shell 时执行一次
$ZDOTDIR/.zshrc    # 交互式 shel 时加载
$ZDOTDIR/.zlogin   # 登陆 shell 时执行
$ZDOTDIR/.zlogout  # 退出登陆时执行
```

如果没有设置 `$ZDOTDIR` 将使用 `$HOME`。

## 终端多路复用器

- screen

  screen 是最初的终端多路复用器，除非登陆环境且不能安装其他多路复用器，否则不应该使用 sreen。

- tmux

  tmux 是一个灵活的终端多路复用器，在 tmux 中有 3 种核心元素交互，从粗颗粒度单元到细颗粒度单元：
  - 会话 session

    一个逻辑单元，可以将其视为特定任务的工作环境，他是其他所有单元的容器

  - 窗口 window

    可以将窗口看作浏览器中的一个标签页，每个会话只有一个窗口

  - 窗格 pane

    这里主要使用的地方，内部运行一个 shell 实例。
    窗格是窗口的一部分，可以水平或垂直分割他，并根据需要进行展开和折叠。

```bash
tmux new -s test
```

上面命令创建一个名为 test 的新会话，tmux 使用 `Ctrl-b` 作为默认的前置键，也称前缀触发器。
可以使用 `Ctrl-b d` 暂时分离当前会话，回到终端系统，使用 `tmux ls` 查看所有会话，之后使用 `tmux attach -t 会话名` 来回到任意一个会话。

注意这和 `Ctrl-z` 完全不同，`Ctrl-z` 会将当前进程挂起，冻结进程并返回到 shell，而 tmux 的 `Ctrl-b d` 会将会话置于后台运行，不会中断。

要列出 tmux 的所有会话窗口，使用 `Ctrl-b w`，这条命令会列出一个列表选择窗口。

下面是常用命令表，trigger 默认是 `Ctrl-b`

会话相关命令

| 任务         | 命令                             |
| :----------- | :------------------------------- |
| 新创建       | `:new -s NAME`                   |
| 重命名       | `trigger + $`                    |
| 列出所有会话 | `trigger + s`                    |
| 关闭         | `trigger + d` 或 `:kill-session` |

窗口相关命令

| 任务         | 命令            |
| :----------- | :-------------- |
| 新创建       | `trigger + c`   |
| 重命名       | `trigger + ,`   |
| 切换到       | `trigger + 1…9` |
| 列出所有窗口 | `trigger + w`   |
| 关闭         | `trigger + &`   |

窗格相关命令

| 任务     | 命令          |
| :------- | :------------ |
| 水平分割 | `trigger + "` |
| 垂直分割 | `trigger + %` |
| 切换     | `trigger + z` |
| 关闭     | `trigger + x` |

tmux 是一个很好的选择，但还有一些其他终端多路复用器可供选择。

- tmuxinator: 管理 tmux 会话的元工具

- Byobu: screen 或 tmux 包装器

- Zellij: 自称为终端工作空间，使用 Rust 编写！并超越了 tmux 所提供的功能，包括一个布局引擎和一个强大的插件系统

- dvtm: 将窗口平铺管理的概念引入终端，功能强大，但和 tmux 一样需要学习

- 3mux: 一个使用 Go 语言编写的简单终端多路复用器，易于使用，但没有 tmux 强大

## Scripts

输入 shell 通常将所有东西作为字符串处理，但他确实支持一些复杂数据类型（可能不应该使用 shell 脚本），例如数组。

```bash
# 定义数组
as=('linux', 'macOS', 'Windows')
# 访问第一个元素
echo "${as[0]}"
# 获取数组长度
numberofos = "${#as[0]}"
```

---

流程控制运行在 shell 脚本中进行分支 (if) 或重复 (for, while)

```bash
# 输出 /temp/ 下面文件名
for afile in /tmp/* ; do
    ehco "$afiel"
done

# 范围循环
for i in {1..10}; do
    echo $i
done

# 无限循环
while true; do
    ...
done
```

---

函数可以编写更多的模块化和可重复使用的脚本，必须在使用前定义，因为 shell 是从上到下解释脚本的。

```bash
sayhi() {
    echo "Hi $1, I hope you are well!"
}

sayhi "Jess"
```

函数定义通过 `$n` 隐式传递

---

如果读取，可以从 stdin 中读取用户输入，可以用他来获得运行时的输入。
此外，与其使用 echo，不如使用 printf，它允许对输入进行精细的控制，包括颜色。

```bash
# 从用户输入中读取数值
read name
# 输出上一步读取的值
printf "Hello %s" "$name"
```

通过两步将一个文本文件变成一个可执行的、能被 shell 运行的脚本（这里扩展名并不重要，`.sh` 只是一种惯例）

- 文本文件需要在第一行声明解释器，使用所谓的 shebang: `#!interpreter [arguments]`

- 然后使用 `chomod +x` 来使脚本可执行，或者 750，则符合最小权限原则。

一个可移植的 bash shell 脚本骨架如下：

```bash
#! /usr/bin/env bash
set -o errexit  # 遇到错误时立即退出脚本
set -o nounset  # 遇到未定义的变量时报错
set -o pipefail # 管道中任意命令失败，整个管道都视为失败

firstarguments="${1:-somedefaultvalue}"

echo "$firstarguments"
```

在开发的时候，如果要检查脚本，确保正确使用命令和指令，可以使用 ShellCheck，并考虑 shfmt 格式化代码。
此外，在脚本提交到仓库前，使用 bats 进行基准测试，bats 是 Bash Automated Testing System 的缩写，运行将测试文件定义为带有测试用例特殊语法的 Bash 脚本。
每个用例都是一个带有描述的 bash 函数，通常会调用这些脚本作为 CI 管道的一部分，作为一个 GitHub 动作。

下面是一个获取 GitHub 用户信息的脚本示例

```bash
#!/usr/bin/env bash

set -o errexit
# 控制错误追踪
# - 在函数内部或子 shell 中发生的错误也会触发 ERR 信号
# - 相应的错误处理代码（通过 trap 设置的）会被执行
set -o errtrace
set -o nounset

# Command line parameter
targetuser="${1:-starslayerx}" # 获取第一个参数，默认使用 starslayerx

# Check if your dependencies are met:
if ! [ -x "$(command -v jq)" ]  # 依赖检查：js (JSON 解释器) 是否已安装，-x 检查是否可执行
then
  echo "jq is not installed" >&2
  exit 1
fi

# Main
githubapi="https://api.github.com/users/"
tmpuserdump="/tmp/ghuserdump_$targetuser.json"

# 获取数据
result=$(curl -s $githubapi$targetuser)  # curl -s 静默模式调用 GitHub api
echo $result > $tmpuserdump  # 保存到临时文件

# 数据处理
name=$(jq .name $tmpuserdump -r)  # 从 JSON 提取用户名
created_at=$(jq .created_at $tmpuserdump -r)  # 提取创建时间

# 格式化输出
joinyear=$(echo $created_at | cut -f1 -d"-")  # 格式化日期
echo $name joined GitHub in $joinyear
```

输出如下

```text
Starslayerx joined GitHub in 2019
```

##

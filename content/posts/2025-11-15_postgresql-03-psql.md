+++
date = '2025-11-14T8:00:00+01:00'
draft = true
title = 'Postgresql psql Tool Introduction'
tags = ['PostgreSQL']
+++

psql 是 PostgreSQL 中的一个交互式命令行工具，类似 Oracle 的 sqlplus，它允许用户交互式输入 SQL 语句或命令，然后发送给 PostgreSQL 服务器，再显示结果。
此外，psql 还有大量类似 shell 的特性来编写脚本，实现自动化操作。

在安装 postgresql 的时候，会创建一个与初始化数据库时的操作系统同名的数据库用户，这个用户是这个数据库的超级用户。
因此，在 OS 用户下登陆数据库时，会执行操作系统认证，因此无需用户名和密码，也可以通过修改 pg_hba.conf 文件来要求用户输入密码。

psql 也支持使用命令行参数来查询信息和执行 SQL，这种非交互模式与 linux 命令就没有太大区别了。
例如使用 `psql -l` 查看数据库，当然也可以通过命令进入数据库后，输出 `\l` 命令查看有哪些数据库。
默认会有一个叫 postgres 的数据库，这是默认安装后就有的一个数据库。

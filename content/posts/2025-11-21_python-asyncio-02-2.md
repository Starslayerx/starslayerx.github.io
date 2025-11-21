+++
date = '2025-11-21T8:00:00+08:00'
draft = true
title = 'Python Asyncio 02: Asyncio Basics Part 2'
tags = ['Python', 'Asyncio']
+++

## Tasks, coroutines, furtures, and awaitables

Coroutines 和 tasks 都是 await 表达式，那他们的相同线程是哪个？
下面介绍 `future` 也被称作 `awaitable`，理解 futures 是理解 asyncio 内部工作的重点。

### Introducing futures

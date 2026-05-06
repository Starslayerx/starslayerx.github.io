+++
date = '2026-05-02T8:00:00+01:00'
draft = false
title = 'Learning concurrency - a deep dive into multithreading with Python'
categories = ['Blog']
tags = ['Python']
+++

这篇文章讲 Python 中的并发，例如多线程 *multithreading*, 多进程 *multiprocessing*, 竟态条件 *race conditions* 以及同步状态 *synchronization* 的机制，例如锁。
然后会探讨如何关闭 GIT 来实现 Python 中真正的多线程，并通过清晰的代码来突出差异、优点以及注意事项。

## Introduction

你可能会好奇为什么会需要并发。
大多数情况下，并不需要，但如果是以下情况可能需要：

- 数据处理 *data processing* 和*[ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load)* - 解析大量文本文件、清理混乱的数据、或将复杂的正则表达式应用到数百万行上
- 加密 *cryptography* 和哈希 *hashing* - 读取文件并计算每一行的加密哈希 (例如 SHA-256)
- 数据科学 - 计算蒙特卡罗模拟 *Monte Carlo simulations* 要求大量数学计算，例如 numpy, pandas 或 scikit-learn
- 网络操作 - 下载文件，爬取网站或调用 REST API

但首先，这里先把术语理清楚：

- **Sequential** 串行：单个进程一次只做一件事，在做下一件事情之前会等待直到完成后再开始下一个进程
- **Concurrency** 并发：单个进程同时做多件事，但不一定在同一时刻做
- **Event Loop** 事件循环: 一种控制接口，持续等待事件（例如 I/O 操作、定时触发或用户操作），分派对应的任务，然后重复这个过程
- **Parallelism/Multiprocessing** 并行/多进程: 在同一时刻做多件事
- **Multithreading** 多线程: 这种编程模型是单个进程生成多个独立的执行线程，从而达到并发。所有线程都共享内存空间和资源。

线程安全 *thread safety* 是指程序或系统的一种特性，它允许多个线程同时访问和修改共享内存与资源，而不会造成数据损坏、内存泄漏或程序崩溃。

+++
date = '2025-09-02T8:00:00+08:00'
draft = true
title = 'Code Surgery: How AI Assistants Make Precise Edits to Your Files'
tags = ['LLMs']
+++
原文链接: [代码手术: AI 助手如何对你的文件做出精确改动](https://fabianhertwig.com/blog/coding-assistants-file-edits/)

昨天的文章介绍了如何制作一个基本的 AI 编程助手, 今天更近一步, 探讨 AI 助手如何对文件进行精确的修改.

> 实际的 Coding Agent 一般并不会直接读取文件的全部代码, 因为这样可能会读取大量无关的代码, 导致 LLM 输出质量下降以及浪费 token, 同样的, 昨天的 agent 是直接重写整个文件, 也会导致输出 token 非常多, 一般也是会只对文件中必要的行进行修改, 而不是整个文件.

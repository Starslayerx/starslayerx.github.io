+++
date = '2025-09-11T8:00:00+08:00'
draft = false
title = 'English for Programmers - 01'
categories = ['Blog']
tags = ['English']
+++

# Unit 1. Implementing Code

This unit will cover:

- Use technical verbs to **accurately define tasks** and actoins
- Write **commit messages** in the corret Git format
- Confidently **name** the **symbols** used when writing code
- Understand **vocabulary** for **syntax** and programming rule

## vocaluary - action verbs 动词词汇

He **optimised** the queries to improve the response time.

- **optimised**: 提高 (improved)

Can you **implement** the new feature we discussed yesterday?

- **implement**: 实现 (put int action)

The team will **integrate** a third-party API to get real-time data.

- **intergrate**: 整合 (combine)

As our user base grows, we'll need to **scale** our infrastructure.

- **scale**: 扩展 (increase capacity)

Have you had a chance to **refactor** the code yet?

- **refactor**: 重构 (change)

The process it taking too long. How can we **streamline** it?

- **straemline**: 简化 (simplify)

Let's **execute** the script before we go for lunch.

- **execute**: 执行 (run)

The settings haven't been **configured** yet.

- **configured**: 配置 (set up)

> Note: optimise spelling  
> British English '-ise' vs. Amercian English '-ize'

## grammar - imperative present tense 语法: 祈使现在时

- Imperative 祈使语气: 用来下达命令、发出请求、给予指示或建议的语气. 核心功能是告诉某人做某事.
- Present Tense 现在时: 这里的"现在时"并不是指描述现在正在发生的动作, 而是指这个动词形式

For readability and consistency in commit messages within a team, Git recommends using the **Impreative Present Tense**.  
Git 建议使用祈使命令式编写提交信息, 编写时将其看成给版本控制和其他开发者看的命令.

> Tip: message 应该描述这个修改将实现的功能, 而不是已经编写的功能.  
> 下面是一个例子:

|                                 Recommended                                 |                                   Not Recommended                                    |
| :-------------------------------------------------------------------------: | :----------------------------------------------------------------------------------: |
| Add new feature for user authentication. Resolve issue with data validation | Added a new feature for user authentication. Resolved the issue with data validation |

> 当编写 imperative 句子的时候, 可以省略 a/an/the

### Keyboard Symbols 键盘符号

想象一下, 你在一个团队中合作, 当他们查看你的代码并给予建议的时候, 某人说:  
"Can you try replacing the **asterisk** with an **ampersand** and adding a **tilde** after the **pipe**?"

Me: ???

| Symbol |        English        |
| :----: | :-------------------: |
|  `!`   |    exlamation mark    |
|  `#`   |         hash          |
|  `^`   |         caret         |
|  `&`   |       ampersand       |
|  `*`   |       asterisk        |
|  `(`   | bracket / parentheses |
|  `~`   |         tilde         |
|  `\|`  |         pipe          |
|  `\`   |       backslash       |
|   \`   |       backtick        |

| Symbol |        English         |
| :----: | :--------------------: |
|  `"`   |      double quote      |
|  `'`   |      single quote      |
|  `/`   |     forward slash      |
|  `:`   |         colon          |
|  `;`   |       semicolon        |
|  `<`   |     angle bracket      |
|  `,`   |         comma          |
|  `{`   | curly bracket / braces |
|  `[`   |     square bracket     |
|  `_`   |       underscore       |
|  `-`   |         hypen          |

1. _Kebab case_ is a naming convention where all letters are lowercase and words are sperated by **hypen**.  
   e.g. `my-variable`

2. _Sanke case_ is a naming convention where all letters are lowercase and words are separated by **underscore**.  
   e.g. `my_variable`

3. Many programming languages ues **single qoutes** or **double qoutes** to denote strings.  
   e.g. `"my variable"`

4. HTML tags are enclosed in **angle brackets**.  
   e.g. `<div>`

|            常见命名风格             |    形式示例    |                    常见场景                     |                      特点 / 备注                      |
| :---------------------------------: | :------------: | :---------------------------------------------: | :---------------------------------------------------: |
|        **camelCase 驼峰式**         | `userProfile`  |    JavaScript 变量、函数名; Java、C# 方法名     |            首字母小写，后续单词首字母大写             |
|        **PascalCase 大驼峰**        | `UserProfile`  |    类名(Java、C#、TypeScript)、组件名(React)    |                  每个单词首字母大写                   |
|       **snake_case 蛇形命名**       | `user_profile` |         Python 变量/函数名; 数据库字段          |                单词用 `_` 分隔, 全小写                |
| **SCREAMING_SNAKE_CASE 全大写蛇形** |  `MAX_VALUE`   |              常量(C、Python、Java)              |                    全大写 + 下划线                    |
|       **kebab-case 烤肉串式**       | `user-profile` |     URL、CSS 属性、CSS 类名、配置项、文件名     | 单词用 `-` 分隔, 全小写. 不能当变量名(`-` 被视为减号) |
| **Train-Case 标题式 / Header-Case** | `User-Profile` | 文档标题、部分配置(HTTP Header: `Content-Type`) |            类似 PascalCase, 但用 `-` 分隔             |
|            **dot.case**             | `user.profile` |  部分配置文件、键路径(MongoDB、Elasticsearch)   |                    用 `.` 分隔单词                    |
|           **Space Case**            | `User Profile` |                UI 文本、自然语言                |             单词直接空格分隔(不用于代码)              |

### listening syntax

你的朋友正在为你介绍一门新语言的语法 syntax (rule defining the structure of the symbols, punctuation and words of a programming language)

音频在[这里](https://drive.google.com/file/d/1nJ5_EXkGJpP-LA9sIi_9u_fKWGEbYMeM/view)

文本如下

```
Let me explain the syntax of my programming language.

So firstly, variable names aren't case sensitive and only contain alphanumberic characters.
Secondly, comments can be denoted using an ampersand at the beginning.
Thirdly, indentation is not mandatory but it is encouraged for readability.
Fourthly, function names must start with a verb and be descriptive of purpose.
And finally then, mathematical symbols are not allowed to be used.
```

词汇:

- case sensitive: 大小写明感
- alphanumberic: 字母数字
- denote: 代表
- ampersand: &
- indentation: 缩进
- mandatory: 强制的
- descriptive 描述的

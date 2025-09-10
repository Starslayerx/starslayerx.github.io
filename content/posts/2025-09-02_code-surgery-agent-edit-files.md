+++
date = '2025-09-02T8:00:00+08:00'
draft = false
title = 'How AI Assistants Make Precise Edits to Your Files'
tags = ['LLMs', 'Agent']
+++

之前的文章介绍了如何制作一个基本的 AI 编程助手, 今天更近一步, 探讨 AI 助手如何对文件进行精确的修改.

> 实际的 AI Agent 不会读取所有的项目代码, 一般只会读取当前文件的代码, 当需要时才会去读取相关的代码文件. 然而, 输出也不会输出要修改的整个文件的代码, 因为这样输出不仅很慢, 同时成本也很高, 会有大量重复代码导致浪费(以 deepseek 为例, 输出 token 的价格是输入 token 价格的3倍, 是缓存命中输入 token 价格的24倍!), 因此一般是让模型输出要修改的代码和修改后的代码.  

> 既然不能一次输出文件的所有代码, 这就引出了一个问题: 如何精确的修改代码文件? 首先要确定一种让模型准确地描述修改的格式, 并且提供健壮的格式匹配与错误重试机制(模型输出代码可能少个空格或者Tab), 这篇文章对这个问题做了探讨.

将 AI agent 生成的代码直接修改到文件中是一项核心能力, 然而实际上这常常出乎意料的困难, agent 可能会提出一个代码修改方案, 但实际修改却失败, 例如"找不到匹配的上下文"之类的错误, 需要手动干预. 许多 AI 编程助手的开发者都遇到过这种情况, 虽然 AI 理解代码的意图, 但将这种理解转化为精确的文件修改却带来了重大的技术挑战.


### Why Precise File Editing Matters 为什么精确的文件编辑至关重要
有效的文件编辑是编程助手的价值核心, 如果其不能可靠的修改文件, 需要人为手动修改, 就退化成了 AI 聊天引擎, 相比之下, 一个能够可靠自动化编辑的助手可以为开发者节省大量时间和认知负担.  
根本的挑战在于, LLM 缺乏直接的文件系统访问权限, 他们必须通过专门的工具来描述预期的修改, 然后这些工具或 API 解释指令并尝试执行, LLM 的描述与文件系统状态之间的这种交接是常见的问题来源.  
使用 GitHub Coplit、Aider、RooCode 或 Cursor 等工具的用户可能已经观察到这些问题: 编辑器无法找到正确的插入点、缩进不正确, 或者工具最终请求手动应用.  

### What Will Cover 本文内容
本文将探讨几种编程助手系统的文件编辑机制: Codex、Aider、OpenHands、RooCode和Cursor. 对于开源系统(Codex、Aider、OpenHands、RooCode), 本文提供的见解来源于分析它们各自的代码库. 于闭源的Cursor, 见解则来自公开讨论及其团队的访谈.  
对于每个系统, 将分析:  
- 它如何从AI接收编辑指令
- 它如何解释和处理这些指令
- 它如何将更改应用到文件
- 它如何处理错误和边缘情况
- 它如何提供结果反馈

理解这些机制有助于深入了解自动化代码编辑的困难以及不同系统采用的日益复杂的解决方案.

### Key Concepts in AI Coding Agent 编程助手的核心概念
在继续之前, 先定义一些该领域常用的术语:
- Patch 补丁: 文件更改(添加、删除)的正式规范, 通常包含元数据, 如文件路径和应用的上下文.
- Diff 差异: 一种突出显示文本版本间逐行差异的范式, 通常使用 `+` 和 `-` 标识符, 侧重与内容变更
- Search/Replace Block 搜索/替换块: 使用一种分隔符(例如 `<<<<<<<` SEARCH, `>>>>>>>` REPLACE)
- Context Lines 上下文行: 包含在 Patch 或 Diff 中围绕更改的未修改行, 用于精确定位修改点
- Hunk 块: Patch 或 Diff 中的连续修改块, 包含上下文行和修改
- Fuzzy Matching 模糊匹配: 用于查找文本字符串近似匹配的算法(例如 Levenshtein 距离), 处理微小差异
- Indentation Preservation 缩进保留: 在文件编辑其间保持一致的空白缩进(Space、Tab), 对语法和可读性至关重要
- Fench 围栏: 分隔符(例如\`\`\`), 用于在文本或指令中清晰标记代码块的边界

### The File Editing Workflow 文本编辑流程
大多数 AI 代码编辑系统遵循一个通用流程:  
```
LLM(生成变更描述) → 工具(解析和应用) → 文件系统(状态变更) → 反馈(工具返回结果) → LLM(处理反馈)
```
上面流程中有这些挑战:  
#### 1. Locating the Edit Target 定位编辑目标
LLM 通常基于目标文件可能已过时, 或不完整的视图进行操作, 在以下情况很难找到预期的编辑位置:
- LLM 上次访问后, 文件已修改
- 文件包含多个相似的代码片段
- 文件超出了 LLM 的上下文窗口量

当文件状态出现分歧时, 上下文不匹配很常见, 健壮的系统提供详细的错误反馈, 使LLM能够适应.

#### 2. Handling Multi-File Changes 处理多文件变更
代码修改经常涉及多个文件, 这引入了复杂性:  
- 确保相关编辑的一致性
- 管理文件之间的依赖关系
- 以正确的顺序应用修改

大多数系统通过按文件顺序处理编辑来解决这个问题

#### 3. Maintaing Code Style 保持代码风格
开发者要求遵守特定的格式约定, 自动化编辑必须保留:
- 缩进风格 (空格/宽度)
- 行尾约定
- 注释格式
- 一致的间距模式

#### 4. Managing Failures 管理错误
一个健壮的编辑系统应该优雅地处理失败:  
- 提供清晰的错误原因解释
- 提供诊断信息以帮助纠正
- 在初始失败时尝试替换策略

#### Common Edit Description Formats 常见的编辑描述格式
AI 系统使用多种格式来传达预期的更改:
- Patchs 补丁: 详细的添加/删除指令, 通常基于标准补丁格式
- Diffs 差异: 显示原始状态和期望之间的差异
- Search/Replace Blocks 查询/搜索块: 明确定义于查找/替换操作
- Line Operations 行操作: 按行号指定编辑(不太常见, 效果很差)
- AI-Assisted Application AI辅助应用: 使用辅助AI模型专门应用复杂的更改

下面看看具体系统如何实现这些概念

### Codex: A Straightforward Patch-Based System 一个简单的基于补丁的系统
OpenAI 的 Codex CLI 使用一个相对简单、结构化的补丁格式, 其有效性部分源于 OpenAI 能够专门训练其模型以可靠地生成这种格式.

#### The Codex Patch Foramt | Codex 补丁格式
LLM 使用以下格式表达修改
```
*** Begin Patch
*** [Operation] File: [filepath]
@@ [text matching a line near the change]
  [context line (unchanged, starts with space)]
- [line to remove (starts with -)]
+ [line to add (starts with +)]
  [another context line]
*** End Patch
```
关键特性:
- `Operation`: 添加文件(Add File)、更新文件(Update File)或删除文件(Delete File)
- `@@`: 后面跟着编辑位置附近一行的文本内容(例如函数/类定义), 用于定位更改, 避免了对行号的依赖
- `Context lines`: 上下文行, 以空格开头, 必须匹配现有文件内容且保持不变, 用于精确定位.
- 以 `-` 开头的行: 标记为要删除
- 以 `+` 开头的行: 标记为要添加

如下面这个例子
```
*** Begin Patch
*** Update File: main.py
@@ def main():
   # This is the main function
-  print("hello")
+  print("hello world!")
   return None
*** End Patch
```
这里, `@@ def main():` 帮助定位函数, 而以空格开头的上下文行(`# This is ...` 和 `return None`) 精确定位了确切的编辑位置  
系统尝试精确匹配 `@@` 行和上下文行, 如果失败, 则采用回退策略: 首先尝试删除行尾后的匹配, 然后尝试删除所有空白后的匹配, 这种灵活性考虑到了 LLM 视图与实际文件之间的微小差异, 一个补丁可以包含多个 `@@` 部分以定位文件的不同部分.  

#### Patch Parsing and Application 补丁解析和应用
通过 `apply_patch` 工具接收到补丁后, 系统执行以下步骤:  
1. 验证补丁结构是否正确 (`*** Begin Patch` / `*** End Patch`)
2. 找到目标文件
3. 加载目标文件的当前内容
4. 将补丁解析为离散的操作
5. 尝试将更改应用到加载的文件内容中

#### Fuzzy Matching for Robustness 健壮的模糊匹配
对上下文的渐进式匹配策略增加了健壮性:
1. 尝试精确匹配
2. 如果失败, 尝试忽略行尾的匹配
3. 如果失败, 尝试忽略所有的空白匹配

这有助于模型预期的文件内容, 和实际文件内容的微小差异

#### Error Handing and Feedback Mechanisms 错误匹配和反馈机制
Codex 在失败时提供结构化的 JSON 反馈, 帮助 LLM 进行纠正尝试

- 上下文不匹配 (清晰地指出了不匹配和不同的行)
    ```JSON
    {
      "exit_code": 1,
      "stderr": "Error: context line 3, \"  const response = await fetch(`/api/users/${userId}`);\" does not match \"  const response = await fetch(`/api/users/${userId}`, { headers });\""
    }
    ```

- 文件未找到 (明确的文件访问错误)
    ```JSON
    {
      "exit_code": 1,
      "stderr": "Error: File not found: src/api.js"
    }
    ```

- 无效的补丁格式 (特定的格式违反反馈)
    ```JSON
    {
      "exit_code": 1,
      "stderr": "Error: Patch must start with *** Begin Patch"
    }
    ```

这种详细的结构化反馈允许 LLM 诊断失败, 并可能修正后重试

#### Example: Real-World Edit Scenario 示例: 真实世界的编辑场景
下面来看看 Codex 如何处理一个真实的编辑场景, 假设要向一个获取用户数据的函数添加错误处理

`src/api.js` 中的原始代码:
```Javascript
async function fetchUserData(userId) {
    const response = await fetch('/api/users/${userId}');
    const data = await response.json();
    return data;
}
```

LLM 生成一个补丁:
```
*** Begin Patch
*** Update File: src/api.js
@@ async function fetchUserData(userId) {
-  const response = await fetch(`/api/users/${userId}`);
-  const data = await response.json();
-  return data;
+  try {
+    const response = await fetch(`/api/users/${userId}`);
+    if (!response.ok) {
+      throw new Error(`Failed to fetch user data: ${response.status}`);
+    }
+    const data = await response.json();
+    return data;
+  } catch (error) {
+    console.error(`Error fetching user ${userId}:`, error);
+    throw error;
+  }
}

@@ function formatUserData(data) {
-  return data;
+  return {
+    id: data.id,
+    name: data.name,
+    email: data.email,
+    formattedDate: new Date(data.createdAt).toLocaleDateString()
+  };
}
*** End Patch
```
这个例子显示了一个修改同一文件中两个不同函数的补丁, 每个函数都有自己的 `@@` 上下文标记

#### OpenAI's Patch Format Standardization | OpenAI 的补丁格式标准化
随着 GPT-4.1 (2025-04) 的发布, OpenAI 发布了一个 "[Prompt Cookbook](https://cookbook.openai.com/examples/gpt4-1_prompting_guide#appendix-generating-and-applying-file-diffs)", 详细介绍了这种推荐的补丁个似乎和一个参考实现(`apply_patch.py`). 他们指出对 GPT-4.1 在此格式上进行了大量训练, 这有助于其在 Codex CLI 生态系统中的有效使用.  
OpenAI 的评论强调, 成功的格式通常避免使用行号, 并使用不同的分隔符清晰地提供要替换的代码及其替换内容, 这为可靠的 AI 驱动编辑提出了核心原则.  


### Aider: A Multi-Format Editing System | Aider: 一个多格式编辑系统
Aider 采用了一一种更灵活的方法, 支持多种编辑格式, 可以选择最合适任务或特定 LLM 的格式.  

#### Pluggable Edit Format Architecture  可插拔的编辑格式架构
Aider 使用一个"编码器" [Coder](https://github.com/Aider-AI/aider/blob/e4fc2f515d9ed76b14b79a4b02740cf54d5a0c0b/aider/coders/base_coder.py#L88) 类系统, 每个类负责处理特定的编辑格式:
```Python
class Coder:
    # ... 
    def get_edits(self): # 将 AI 响应解析为编辑操作
        raise NotImplementedError

    def apply_edits(self, edits): # 将解析后的编辑应用到文件
        raise NotImplementedError
```

#### Supported Edit Formats in Aider | Aider 中支持的编辑格式
Aider 指出多种格式, 根据模型或用户配置 (`--edit-format`) 进行选择
- EditBlock Format (Search/Replace) 编辑块格式: 直观的格式, 清晰显示搜索/替换块
    ```
    file.py
    <<<<<<< SEARCH
    # Code block to find
    =======
    # Code block to replace with
    >>>>>>> REPLACE
    ```

- Unified Diff Format(udiff) 统一差异格式: 标准差异格式(`diff -U0` 风格), 适用于复杂更改
    ```
    --- file.py
    +++ file.py
    @@ -10,7 + 10,7 @@
    - return "old value"
    + return "new value"
    ```

- OpenAI Patch Format | OpenAI 补丁格式: Aider 实现了 OpenAI 的参考格式, 利用了 GPT-4.1 再此语法上的训练
    ```
    *** Begin Patch
    *** Update File: file.py
    @@ class MyClass:
        def some_function():
    -        return "old"
    +        return "new"
    *** End Patch
    ```

- 其他格式:
    - `whole`: LLM 返回完整的修改后文件内容, 简单, 但对于大文件可能效率地下.
    - `diff-fenced`: 差异格式变体, 文件名在代码围栏内, 用户 [Gemini](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/tools/edit.ts#L37) 等模型
    - `editor-diff` / `editor-whole`: 用于特定内部模式的简化版本

#### Flexible Search Strategies 灵活的搜索策略
当应用搜索/替换块时, Aider 尝试按顺序使用多种匹配策略:
1. 精确匹配
2. 空白不敏感匹配
3. 保留缩进的匹配
4. 使用 `difflib` 进行模糊匹配

这种分层方法增加了即使 SEARCH 块存在微小缺陷, 也能成功修改的可能性

#### Detailed Error Reporting 详细的错误报告
Aider 在编辑失败时擅长提供信息丰富的反馈
```
# 1 SEARCH/REPLACE block failed to match!

## SearchReplaceNoExactMatch: This SEARCH block failed to exactly match lines in src/api.js
<<<<<<< SEARCH
async function fetchUserData(userId) {
  const response = await fetch(`/api/users/${userId}`);
  const data = await response.json();
  return data;
}
=======
...
>>>>>>> REPLACE

Did you mean to match some of these actual lines from src/api.js?

async function fetchUserData(userId) {
    const response = await fetch(`/api/users/${userId}`);
    // Some comment here
    const data = await response.json();
    return data;
}

The SEARCH section must exactly match an existing block of lines including all white space, comments, indentation, docstrings, etc

# The other X SEARCH/REPLACE blocks were applied successfully.
Don't re-send them.
Just reply with fixed versions of the blocks above that failed to match.
```
这种反馈比简单的失败消息要详细得多, 它解释了不匹配的原因, 建议了潜在的正确目标, 重申了匹配规则, 并指示 AI 如何继续, 这种详细的指示极大地提高了 AI 纠正错误修改的能力.  
在采用 OpenAI 格式的同时, Aider 通过更大的灵活性和更丰富的错误处理对齐进行了增强.

### OpenHands: Blending Traditional and AI-Assisted Editing / 融合传统和AI辅助编辑
OpenHands 主要依赖传统的编辑应用方法, 同时也包括一个可选的基于 LLM 的编辑功能

#### Traditional Edit Application 传统编辑应用
OpenHands 主要使用传统的编辑方法, 内置支持检查不同的补丁格式, 包括: 统一差异(unified diffs)、git 差异、上下文差异(context diffs)、ed 脚本和 RCS ed 脚本(使用正则表表达式). 该系统支持几种传统的编辑方法:  
- 字符串替换
- 基于行的操作(按行号)
- 标准补丁应用工具

包括注入空白规范化之类的功能, 以处理补丁缩进的变化

#### Optional LLM-Based Editing Feature 可选的基于LLM的编辑功能
OpenHands 允许配置一个单独的"草稿编辑器" LLM, 用于一个独特的编辑工作流程:
1. 目标识别: 主 LLM 指定要编辑的目标行范围
2. 内容提取: 工具提取这个特定的代码段
3. LLM 重写: 提取的片段和所需要更改的描述被发送到专门的"草稿编辑器" LLM, 这个编辑器 LLM 可以有不同的配置(模型、温度), 针对编辑器进行了优化
4. 文件重建: 工具从编辑器 LLM 接收修改后的片段, 并将其集成回收文件中, 替换原始行

为了确保草稿编辑器 LLM 产生正确的输出以供集成, 它会收到一个特定的系统提示, 指示它:
- 生成修改后代码段的完整且准确的版本
- 如果某些部分需要保留, 则将任何占位符注释(如 `# no changes needed before this line`) 替换为原始段中的实际未更改的代码
- 确保在整个块中保持正确且一致的缩进
- 输出最终的、完整的编辑目录, 并精确地包括在指定的代码块中, 以便工具轻松解析

使用单独编辑器的 LLM 潜在好处:  
- 特定任务调优: 专门为代码修改优化参数
- 模型灵活性: 对推理和编辑使用不同的模型
- 聚焦提示: 向编辑器 LLM 提供特定的编辑提示词

重建过程仔细地组合了编辑前的内容、LLM 编辑的块和编辑后的内容, 可以执行可选的验证步骤, 如代码检查(linting). 这种基于 LLM 的编辑似乎是 OpenHands 中的一个可选的、可能是实验性的功能, 通常默认禁用.

### RooCode: Advanced Search and Format Preservation / 高级搜索和格式保留
RooCode 使用搜索/替换块格式, 其优势在于用于定位目标块的高级搜索算法, 以及在替换过程中对应代码格式的细致处理.

#### Advanced Search Strategy: Middle-Out Fuzzy Matching | 高级搜索策略: 由内而外的模糊匹配
当搜索块的精确匹配失败时, RooCode 通过其 `MultiSearchReplaceDiffStrategy` 采用"由内而外" (middle-out) 的模糊匹配方法  
1. 估计区域: 在预期位置附近开始搜索 (可能由行号提示)
2. 扩展搜索: 从这个中心点向外搜索
3. 评分相似度: 使用 Levenshtein 距离等算法对搜索块与文件中潜在匹配项之间的相似度进行评分
4. 选择最佳匹配: 选择超过定义阈值的最高分配项

这种策略对于大文件或行号略有不准的情况非常有效, 针对微小的上下文便宜提供了鲁棒性.

#### Emphasis on Indentation Preservation 强制缩进保留
不正确的缩进是编辑文件常见的问题, RooCode 实现了一个复杂的系统来保留格式:  
1. 捕获原始缩进: 记录原始文件中匹配行的确切前导空白 (空格/指标符)
2. 分析相对缩进: 计算替换块中每行相对于其第一行或周围块的缩进
3. 应用原始样式并保持相对结构: 重写应用捕获的原始缩进格式, 同时保持替换代计算出的相对缩进结构

这种缩进的细致关注对于代码可读性和语法正确性(尤其是 Python 等语言)至关重要  
RooCode 编辑过程示例
```
<<<<<<< SEARCH
:start_line:10
-------
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item, 0);
}
=======
function calculateTotal(items) {
  // Add 10% tax
  return items.reduce((sum, item) => sum + (item * 1.1), 0);
}
>>>>>>> REPLACE
```
RooCode 解析 diff:  
- 提取起始行 (10)
- 提取搜索块 (`function calculateTotal...`)
- 提取替换块 (`function calculateTotal...`)

RooCode 应用 diff:  
- 读取文件的当前内容
- 使用模糊匹配找到搜索块的最佳匹配
- 应用替换并保留适当的缩进
- 显示差异视图供用户批准
- 如果批准则应用修改

反馈给 LLM:  
- 如果成功: "更改已成功应用到文件"
- 如果失败: 带有失败原因的详细错误信息

RooCode 将强大的模糊匹配与对维护代码格式完整性的强烈关注相结合


### Cursor: Specialized AI for Change Application / 专注于变更应用的AI
当其他系统改进编辑格式或匹配算法时, Cursor 引入了一个专门的 AI 模型, 专门用于编辑过程的应用步骤.  
这直接解决了一个观察结果: 即使强大的 LLM 擅长代码生成和推理, 也可能难以生成格式完美、位置精确且可以通过简单算法干净应用的差异, 尤其是在复杂文件中.  
Cursor 的方法设计两步 AI 过程:  
1. Sketching 草图: 一个强大的主 LLM 生成预期的修改, 专注于核心逻辑, 而不是完美的差异算法, 这可能是代码的一个粗略描述
2. Applying 应用: 一个单独的、经过订制训练的 "Apply 应用" 模型接收这个草图. 这个专门的模型经过训练, 可以智能地将草图集成到现有的代码库中, 处理上下文、结构的细微差别, 以及输入草图中潜在的不完美之处. 它执行的操作不是简单的文本匹配, 旨在实现智能的代码集成.  

这种策略将高级别的变更生成与详细的应用机制分离开来, 主 LLM 专注于更改什么, 而专业的 Apply 模型专注于如何将更改健壮且准确的集成到文件系统中.  
这里可以看到 Cursor 团队讨论这种[方法](https://youtu.be/oFfVt3S51T4)

#### Evolution and Convergence of Edit Formats 编辑格式的演进与融合
检查这些系统揭示了格式开发中的有趣模式:
- Search/Replace Lineage 搜索/替换体系: Aider 的 EditBlock 格式 ( `<<<<<<<` / `>>>>>>>` ) 建立了一种直观的方法, 后来被 Cline 采用, RooCode 又在此基础上构建
- OpenAI Patch 的影响: 随着 GPT-4.1 发布的特定补丁格式由于集中的模型训练而获得关注, Codex 原生使用它, 也被 Aider 作为一个选项采用
- Underlying Priciples 底层原则: 尽管起源不同, 但成功的格式都趋同 OpenAI 指出的关键思想: 避免使用行号并清晰分隔原始代码和替换代码, 这些特性似乎是可靠的 AI 驱动编程的基础.

### Conculsion and Key Learnings 结论于关键要点
研究 AI 编程助手如何编辑文件揭示了设计复杂技术和演进策略的复杂过程.  
关键要点:
1. 格式很重要: 避免使用行号, 并清晰分隔前后代码的格式是普遍有效的, 尤其是在模型经过训练的情况下.
2. 健壮的匹配关系很重要: 成功的系统采用分层匹配策略(先精确再逐渐模糊), 以在精度和处理微小差异的能力之间取得平衡.
3. 缩进完整性能至关重要: 仔细保留空白和缩进, 对于代码正确性和开发者接受度至关重要
4. 信息丰富的反馈支持纠正: 详细的错误消息, 对于 AI 诊断和修复失败的编辑至关重要
5. 专业化展示前景: 使用专门的 AI 模型来处理特定的子任务, 如变更应用, 代表了一种提高可靠性的高级方法

### Considerations for Tool Builders 给工具构建者的考虑因素
开发健壮的 AI 编辑工具涉及几个考虑因素:
- 实施分层匹配: 从严格匹配开始, 并添加回退的模糊策略
- 优先考虑缩进保留: 投入精力准确维护格式
- 设计可操作的错误反馈: 提供具体、信息丰富的错误消息
- 利用现有格式和实现: 考虑已建立的格式, 并研究开源系统

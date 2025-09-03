+++
date = '2025-09-01T8:00:00+08:00'
draft = false
title = 'Make an AI Coding Agent in python'
tags = ['LLMs']
+++
这篇文件介绍如何使用 Python 制作一个基础的 AI 编程助手

## Minimal AI Coding Agent
下面是一个 AI Coding Agent 至少需要的功能
1. Chat loop 对话循环
2. Call an LLM  调用大语言模型
3. Add tools to call 增加工具调用
4. Handle tool request 处理工具调用请求

### Step 1: Chat Loop
首先, 聊天循环一直循环等待用户输入, Python 的 "input" 方法可以获取用户输入
```Python
print("Type q to quit")

while True:
    user_message = input("You: ")
    if user_message == "q":
        break
    ai_message = f"You said {user_message}... so insightful"
    print(ai_message)
```
目前主流的 LLM 都是无状态的, 所以需要我们手动的去管理对话上下文, 这里使用一个 `fake_ai` 函数模拟真实的 LLM 调用, 并包含 `role` 和 `content` 内容
```Python
# step1.py
import requests

def fake_ai(message):
    latest_user_messages = messages[-1]["content"]
    ai_message = f"You said {latest_user_messages}... so insightful"
    return {
        "role": "assistant",
        "content": ai_message,
    }

print("Press q to quit")
messages = []

while True:
    user_message = input("You: ")
    if user_message == "q":
        break
    messages.append({
        "role": "user",
        "content": user_message,
    })

    ai_message_obj = fake_ai(messages)
    print("AI: " + ai_message_obj["content"])

    messages.append(ai_message_obj)
```

### Step2: Call an LLM
现在聊天循环已经设置好了正确的 API, 下面创建 `llm` 函数来替换 `fake_ai`.
```Python
# step2.py
def call_llm(messages):
    headers = {
        "Authorization": f"Bearer $DASHSCOPE_API_KEY",
        "Content-Type": "application/json",
    }

    data = {
        "model": "qwen-plus",
        "messages": messages,
    }

    base_url = "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions"

    try:
        response = requests.post(url, json=data, headers=headers)
        message = response.json()["choices"][0]["message"]
        return message
    except Exception as e:
        return {"content": f"Error: {e}"}

while True:
    # ...
    ai_message_obj = call_llm(messages)
    # ... unchanged
```

### Step3: Add tools to call
哪些工具是需要的? 一个AI编码代理应该能够读取代码文件  
让我们从一个 `read_file` 工具开始. 我们要定义这个工具以及其参数, 然后将所有可用工具列表传递给 LLM, 当 LLM 认为某个响应需要调用工具, 它会返回一个内容为 `None` 的响应, 和一个要调用的工具列表.  
```
# step3.py

TOOL_SPECS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read the contents of a file",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "The file path to read"
                    }
                },
                "required": ["path"]
            }
        }
    }
]

def call_llm(messages):
    data = {
        "tools": TOOL_SPECS,
        "tool_choice": "auto",
    }

    try:
        print("Full message")
        print(message)
        return message
```
现在, 已经完成了将工具传递给 LLM 的部分, 但是还没有处理工具调用

### Step4: Handle tool requests
虽然 LLM 可以请求调用, 但是需要程序来调用代码以添加上下文或执行操作. LLM 通常会向用户返回一个包含所有新增上下文的最终消息. 在这里需要:  
- A: 检查是否存在 `tool_calls` 键
- B: 将这条调用消息添加到消息列表
- C: 调用用过, 并将调用结果添加到消息列表
- D: 最后一次调用 LLM, 并传入所有的上下文
```
# step4.py
def handel_tool(tool_call):
    # TODO: need to call a read read_file function
    content = "This secret is diving deep"
    return {
        "role": "tool",
        "tool_call_id": tool_call["id"],
        "content": content,
    }

while True:
    # ... unchanged
    ai_message_obj = call_llm(messages)
    
    # A: 判断 AI 是否想调用工具
    if "tool_calls" in ai_message_obj and ai_message_obj["tool_calls"]:
        # B: 添加带有工具调用的 AI 信息
        messages.append(ai_message_obj)

        # C: 执行每个工具, 并获取结果
        for tool_call in ai_message_obj["tool_callls"]:
            tool_result = handle_tool(tool_call)
            messages.append(tool_result)

        # D: 获取 AI 的响应
        final_response = call_llm(responses)
        print(f"AI: {final_response["content"]}")
        messages.append(final_response)
    else: # 默认流程
        print(f"AI: {ai_message_obj["content"]}")
        messages.append(ai_message_obj)
```
下面是工具函数, 以及执行工具的函数
```
# step4_1.py
def read_file(path):
    """读取 path 路径文件内容"""
    try:
        with open(path, "r") as f:
            content = f.read()
    except Exception as e:
        return f"Error reading file: {str(e)}"

def handle_tool(tool_call):
    """执行一个 tool call 并返回值"""
    tool_name = tool_call["function"]["name"]
    tool_args = json.loads(tool_call["function"]["arguments"])

    print(f"[Exexuting {tool_name}...]")

    if tool_name == "read_file":
        result = read_file(**tool_args)
    else:
        result = f"Unknown tool: {tool_name}"

    return {
        "role": "tool",
        "tool_call_id": tool_call["id"],
        "content": result,
    }
```
恭喜! 现在所有步骤都已经完成了, 聊天循环, LLM 调用, 工具传递给 LLM 调用 并 调用工具.  
现在已经完成了让 AI 读取文件的功能, 然后就可以让 AI 查看我们的代码, 并给出建议.

完整代码实现 (仅依赖 python 标准库运行)
```
import requests
import os
import json

TOOL_SPECS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read the content of a file.",
            "parameters": {
                "type": "object",
                "properties":{
                    "path": {
                        "type": "string",
                        "description": "The file path to read"
                    }
                },
                "required": ["path"]
            }
        }
    }
]

def read_file(path):
    """Read the content of a file"""
    try:
        with open(path, "r") as f:
            content = f.read()
        return content
    except Exception as e:
        return f"Error reading file: {str(e)}"

def handle_tool(tool_call):
    """Execute a single tool call and return the result"""
    tool_name = tool_call["function"]["name"]
    tool_args = json.loads(tool_call["function"]["arguments"])

    print(f"[Executing {tool_name}...]")

    if tool_name == "read_file":
        result = read_file(**tool_args)
    else:
        result = f"Unknow tool: {tool_name}"

    return {
        "role": "tool",
        "tool_call_id": tool_call["id"],
        "content": result,
    }

def call_llm(messages):
    api_key = "sk-..."
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    }

    data = {
        "model": "deepseek-v3",
        "messages": messages,
        "tools": TOOL_SPECS,
        "tool_choice": "auto",
    }

    url = "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions"

    try:
        response = requests.post(url, json=data, headers=headers)
        message = response.json()["choices"][0]["message"]
        return message
    except:
        print("error")
        raise Exception("bad")

def fake_ai(messages):
    latest_user_message = messages[-1]["content"]
    return f"AI: You said {latest_user_message}... so insightful "

print("Press q to quit")
messages = []

while True:
    user_message = input("You: ")
    if user_message == "q":
        break

    messages.append({
        "role": "user",
        "content": user_message,
    })

    ai_message_obj = call_llm(messages)

    # Check if AI wants to use tools
    if "tool_calls" in ai_message_obj and ai_message_obj["tool_calls"]:
        messages.append(ai_message_obj)

        for tool_call in ai_message_obj["tool_calls"]:
            tool_result = handle_tool(tool_call)
            messages.append(tool_result)

        final_response = call_llm(messages)
        print(f"AI: {final_response["content"]}")
        messages.append(final_response)
    else:
        print(f"AI: {ai_message_obj["content"]}")
        messages.append(ai_message_obj)
```
运行示例:
```
% uv run minimal-aiagent.py
Press q to quit
You: 你好, 介绍一下你自己.
AI: 你好！我是一个智能助手，可以帮助你回答各种问题、提供建议或执行一些任务。无论是学习、工作还是日常生活相关的问题，你都可以问我。我没有 实体，但可以通过文字与你交流。请问有什么可以帮你的吗？
You: 读取一下 code_example.cc 文件里面的代码, 解释一下这个文件里面的代码实现了什么?
[Executing read_file...]
AI: `code_example.cc` 文件实现了一个优化的 KMP（Knuth-Morris-Pratt）字符串匹配算法。以下是代码的主要功能和逻辑：

### 1. **算法实现**
- **`getNextval` 函数**：
  生成模式串 `P` 的 `nextval` 数组，用于优化 KMP 算法中的回溯过程。`nextval` 数组在匹配失败时，提供更高效的跳转位置。
  - 通过比较模式串的前缀和后缀，动态更新 `nextval` 数组。
  - 如果字符匹配，继续向后比较；否则，利用 `nextval` 数组跳转。

- **`KMP_optimized` 函数**：
  使用 `nextval` 数组在主串 `T` 中查找模式串 `P` 的首次出现位置。
  - 如果匹配成功，返回模式串在主串中的起始位置；否则返回 `-1`。

### 2. **示例运行**
- **`main` 函数**：
  提供了一个示例：
  - 主串（`text`）：`"ABABABABCABABABABD"`
  - 模式串（`pattern`）：`"ABABABD"`
  - 调用 `KMP_optimized` 进行匹配，并输出结果。

### 3. **优化点**
- 使用 `nextval` 数组进一步减少不必要的比较，相比普通 KMP 算法更高效。

### 4. **输出结果**
运行后会输出匹配结果，例如：
主串: ABABABABCABABABABD
模式串: ABABABD
使用 nextval 数组，匹配成功，起始位置: 9

总结：该文件实现了一个优化的 KMP 字符串搜索算法，适用于高效查找子串位置。
You: q
```

## A Step Further
下面让我们更进一步:
- 使用 agent 为 `write_file` 函数提出修改建议
- 允许 agent 写入名为 `README.md` 的文件
- 允许 agent 写入文件以创建 `list_file` 列出文件的功能
    - 这需要处理链式工具调用, 因为编辑文件是 `read + write` 的操作
    - 对比三个文件需要 `read + read + read` 的操作
    - 编辑一个不熟悉格式的文件可能需要 `read + web_reasearch + list + read + write` 操作

### Let AI suggest the "write file" function
我们可以使用刚刚编写的 agent 来完成写入文件函数的功能:
- 复制一份代码到 `step4_2.py`
- 运行 `minimal-aiagent.py`
- 提示词: `你是一名资深工程师. 请查看 step4_2.py 文件, 如何添加一个 write_file 工具让 agent 修改文件? 要确保编写的工具与文件中的代码类型匹配, 最后展示代码更改.`

示例:
```
% uv run minimal-aiagent.py
Press q to quit
You: 你是一名资深工程师. 请查看 step4_2.py 文件, 如何添加一个 write_file 工具让 agent 修改文件? 要确保编写的工具与文件中的代码类型匹配,  最后展示代码更改.
[Executing read_file...]
AI: 为了添加 `write_file` 工具，我们需要在 `TOOL_SPECS` 中添加一个新的工具定义，并实现 `write_file` 函数。以下是修改后的代码：

import requests
import os
import json

TOOL_SPECS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read the content of a file.",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "The file path to read"
                    }
                },
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "Write content to a file.",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "The file path to write"
                    },
                    "content": {
                        "type": "string",
                        "description": "The content to write"
                    }
                },
                "required": ["path", "content"]
            }
        }
    }
]

def read_file(path):
    """Read the content of a file"""
    try:
        with open(path, "r") as f:
            content = f.read()
        return content
    except Exception as e:
        return f"Error reading file: {str(e)}"

def write_file(path, content):
    """Write content to a file"""
    try:
        with open(path, "w") as f:
            f.write(content)
        return "File written successfully."
    except Exception as e:
        return f"Error writing file: {str(e)}"

def handle_tool(tool_call):
    """Execute a single tool call and return the result"""
    tool_name = tool_call["function"]["name"]
    tool_args = json.loads(tool_call["function"]["arguments"])

    print(f"[Executing {tool_name}...]")

    if tool_name == "read_file":
        result = read_file(**tool_args)
    elif tool_name == "write_file":
        result = write_file(**tool_args)
    else:
        result = f"Unknown tool: {tool_name}"

    return {
        "role": "tool",
        "tool_call_id": tool_call["id"],
        "content": result,
    }

def call_llm(messages):
    api_key = "sk-..."
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    }

    data = {
        "model": "deepseek-v3",
        "messages": messages,
        "tools": TOOL_SPECS,
        "tool_choice": "auto",
    }

    url = "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions"

    try:
        response = requests.post(url, json=data, headers=headers)
        message = response.json()["choices"][0]["message"]
        return message
    except:
        print("error")
        raise Exception("bad")

def fake_ai(messages):
    latest_user_message = messages[-1]["content"]
    return f"AI: You said {latest_user_message}... so insightful "

print("Press q to quit")
messages = []

while True:
    user_message = input("You: ")
    if user_message == "q":
        break

    messages.append({
        "role": "user",
        "content": user_message,
    })

    ai_message_obj = call_llm(messages)

    # Check if AI wants to use tools
    if "tool_calls" in ai_message_obj and ai_message_obj["tool_calls"]:
        messages.append(ai_message_obj)

        for tool_call in ai_message_obj["tool_calls"]:
            tool_result = handle_tool(tool_call)
            messages.append(tool_result)

        final_response = call_llm(messages)
        print(f"AI: {final_response['content']}")
        messages.append(final_response)
    else:
        print(f"AI: {ai_message_obj['content']}")
        messages.append(ai_message_obj)

### 主要变更：
1. **添加 `write_file` 工具定义**：
   - 在 `TOOL_SPECS` 列表中添加了一个新的工具定义，支持 `path` 和 `content` 参数。
2. **实现 `write_file` 函数**：
   - 新增 `write_file` 函数，用于将内容写入文件。
3. **扩展 `handle_tool` 函数**：
   - 在 `handle_tool` 中添加了对 `write_file` 工具的支持。

现在，Agent 可以通过调用 `write_file` 工具修改文件内容。
You: q
```

太棒了! 现在这个 coding agent 已经在帮助我们编写代码和改进代理本身了.  
为了确保 agent 不会错误的修改其他地方的文件, 这里通过保证修改的文件在父目录下面来改进写入文件功能
```Python
# Define safe directory (current directory only)
SAFE_DIR = os.path.abspath(os.getcwd())

def is_safe_path(path):
    """Check if path is within the safe directory"""
    # Resolve the absolute path
    abs_path = os.path.abspath(path)
    # Check if it's within the safe directory
    return abs_path.startswith(SAFE_DIR)

def write_file(path, content):
    """Write content to a file"""
    if not is_safe_path(path):
        return f"Error: Access denied - path outside safe directory"
    try:
        with open(path, "w") as f:
            f.write(content)
        return "File successfully written"
    except Exception as e:
        return f"Error writing file: {str(e)}"
```
**请自行判断风险**

下面冒险运行最新的 `step4_2.py` 代码:
```
% uv run step4_2.py
Press q to quit
You: 向 README.md  文件中写入这段文本: "# AI Agent 编写的这段话"
[Executing write_file...]
AI: The text "# AI Age编写的这段话" has been successfully written to the README.md file.
You: q

% cat README.md
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: README.md
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ # AI Age编写的这段话
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

### Let AI write the "list files" with recursive tool calling
让 AI 编写 `list_files` 方法来递归调用工具  
现在这个 coding agent 能够修改文件了, 它可以为新的 `list_files` 函数提出代码更改建议，并将这些更改写入磁盘.
再次强调, 注意风险.  
- 然而, 修改一个文件需要两个链式工具调用: 1.read 2.write
- 比较三个文件则需要三个链式工具调用: 1.read 2.read 3.read

**重构: 递归工具调用**
```Python
# step4_3.py
def handle_message(messages, ai_message_obj):
    # Check if AI wants to use tools
    if 'tool_calls' in ai_message_obj and ai_message_obj['tool_calls']:
        # Add AI message with tool calls
        messages.append(ai_message_obj)
        
        # Execute each tool and add results
        for tool_call in ai_message_obj['tool_calls']:
            tool_result = handle_tool(tool_call)
            messages.append(tool_result)
        
        # Get final response from AI
        final_response = call_llm(messages)
        print(f"maybe final response: {final_response['content']}")
        if 'tool_calls' in final_response and ai_message_obj['tool_calls']:
            handle_message(messages, final_response)
    else:
        print(f"AI: {ai_message_obj['content']}")
        messages.append(ai_message_obj)
        return
        
while True:
    # ...
    ai_message_obj = call_llm(messages)
    handle_message(messages, ai_message_obj)
```
运行示例:
```
% uv run step4_3.py
Press q to quit
You: step1.py 和 step2.py 这两个文件有什么区别?
[Executing read_file...]
[Executing read_file...]
maybe final response: `step1.py` 和 `step2.py` 的主要区别如下：

1. **工具支持**:
   - `step1.py` 只支持 `read_file` 一个工具。
   - `step2.py` 新增了 `write_file` 工具，支持文件的读取和写入。

2. **函数实现**:
   - `step2.py` 中新增了 `write_file` 函数的实现，用于写入文件内容。

3. **消息处理逻辑**:
   - `step1.py` 在处理工具调用时直接返回最终响应。
   - `step2.py` 引入了 `handle_message` 函数，支持递归处理多轮工具调用，确保所有工具调用完成后才返回最终响应。

4. **代码结构**:
   - `step2.py` 的代码结构更加模块化，将消息处理逻辑封装到了 `handle_message` 函数中，提升了可读性和复用性。

总结：`step2.py` 是 `step1.py` 的扩展版本，增加了文件写入功能，并优化了工具调用的处理逻辑。
```

现在再来编写 `list_files`  
```
% uv run step4_3.py
Press q to quit
You: 阅读 step4_4.py，在 TOOL_SPEC 中为 list_files 添加相应功能，并创建执行该操作的函数。不要删除任何现有功能（例如 read file 和 write file）。你是一名资深工程师，请进行深入思考（ultrathink）。完成后，再次读取该文件以确认所有功能仍然存在：聊天循环、调用 LLM、处理工具调用， 以及所有工具（read file、write file 和新增的 list file）。
[Executing read_file...]
...
[Executing write_file...]
maybe final response: 现在，我将重新读取 `step4_4.py` 文件，以验证所有功能是否完整：
[Executing read_file...]
maybe final response: ### 验证结果：
1. **功能完整性**：
   - 保留了原有的 `read_file` 和 `write_file` 功能。
   - 新增了 `list_files` 功能，其工具规范已添加到 `TOOL_SPECS` 中。
   - 更新了 `handle_tool` 函数以支持 `list_files` 调用。

2. **代码逻辑**：
   - 聊天循环、LLM 调用、工具调用处理均未受影响。
   - 新增的 `list_files` 函数逻辑清晰，能够正确列出目录中的文件。

3. **测试验证**：
   - 可以通过实际调用 `list_files` 工具验证其功能。
   - 现有工具（如 `read_file` 和 `write_file`）仍可正常使用。

### 下一步：
如果需要进一步测试或扩展功能，请随时告知！
You: q
```
上面可以看到, 链式调用工作了! agent 修改后对比代码的变化:
```
% diff --color step4_3.py step4_4.py
42a43,59
>     },
>     {
>         "type": "function",
>         "function": {
>             "name": "list_files",
>             "description": "List files in a directory.",
>             "parameters": {
>                 "type": "object",
>                 "properties": {
>                     "directory": {
>                         "type": "string",
>                         "description": "The directory path to list files from"
>                     }
>                 },
>                 "required": ["directory"]
>             }
>         }
63a81,88
> def list_files(directory):
>     """List files in a directory"""
>     try:
>         files = os.listdir(directory)
>         return {"files": files}
>     except Exception as e:
>         return f"Error listing files: {str(e)}"
>
74a100,101
>     elif tool_name == "list_files":
>         result = list_files(**tool_args)
81c108
<         "content": result,
---
>         "content": json.dumps(result),
148c175
<     handle_message(messages, ai_message_obj)
---
>     handle_message(messages, ai_message_obj)
\ No newline at end of file
```
然后测试新的代码
```
% uv run step4_4.py
Press q to quit
You: 列出所有文件
[Executing list_files...]
maybe final response: 当前目录下的文件有：

- `code_example.cc`
- `step4_4.py`
- `step2.py`
- `README.md`
- `step4_3.py`
- `step1.py`
You: q
```

最后给出完整实现
```
# step4_4.py
import requests
import os
import json

TOOL_SPECS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read the content of a file.",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "The file path to read"
                    }
                },
                "required": ["path"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "Write content to a file.",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "The file path to write"
                    },
                    "content": {
                        "type": "string",
                        "description": "The content to write"
                    }
                },
                "required": ["path", "content"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "list_files",
            "description": "List files in a directory.",
            "parameters": {
                "type": "object",
                "properties": {
                    "directory": {
                        "type": "string",
                        "description": "The directory path to list files from"
                    }
                },
                "required": ["directory"]
            }
        }
    }
]

def read_file(path):
    """Read the content of a file"""
    try:
        with open(path, "r") as f:
            content = f.read()
        return content
    except Exception as e:
        return f"Error reading file: {str(e)}"

def write_file(path, content):
    """Write content to a file"""
    try:
        with open(path, "w") as f:
            f.write(content)
        return "File written successfully."
    except Exception as e:
        return f"Error writing file: {str(e)}"

def list_files(directory):
    """List files in a directory"""
    try:
        files = os.listdir(directory)
        return {"files": files}
    except Exception as e:
        return f"Error listing files: {str(e)}"

def handle_tool(tool_call):
    """Execute a single tool call and return the result"""
    tool_name = tool_call["function"]["name"]
    tool_args = json.loads(tool_call["function"]["arguments"])

    print(f"[Executing {tool_name}...]")

    if tool_name == "read_file":
        result = read_file(**tool_args)
    elif tool_name == "write_file":
        result = write_file(**tool_args)
    elif tool_name == "list_files":
        result = list_files(**tool_args)
    else:
        result = f"Unknown tool: {tool_name}"

    return {
        "role": "tool",
        "tool_call_id": tool_call["id"],
        "content": json.dumps(result),
    }

def call_llm(messages):
    api_key = "sk-..."
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    }

    data = {
        "model": "deepseek-v3",
        "messages": messages,
        "tools": TOOL_SPECS,
        "tool_choice": "auto",
    }

    url = "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions"

    try:
        response = requests.post(url, json=data, headers=headers)
        message = response.json()["choices"][0]["message"]
        return message
    except:
        print("error")
        raise Exception("bad")

def fake_ai(messages):
    latest_user_message = messages[-1]["content"]
    return f"AI: You said {latest_user_message}... so insightful "

print("Press q to quit")
messages = []

def handle_message(messages, ai_message_obj):
    """注意: messages 会被修改"""
    if "tool_calls" in ai_message_obj and ai_message_obj["tool_calls"]:
        # Add AI message with tool calls
        messages.append(ai_message_obj)

        # Execute each tool and add results
        for tool_call in ai_message_obj["tool_calls"]:
            tool_result = handle_tool(tool_call)
            messages.append(tool_result)

        # Get final response from AI
        final_response = call_llm(messages)
        print(f"maybe final response: {final_response['content']}")
        if "tool_calls" in final_response and ai_message_obj["tool_calls"]:
            handle_message(messages, final_response)
    else:
        print(f"AI: {ai_message_obj['content']}")
        messages.append(ai_message_obj)
        return


while True:
    user_message = input("You: ")
    if user_message == "q":
        break

    messages.append({
        "role": "user",
        "content": user_message,
    })

    ai_message_obj = call_llm(messages)
    handle_message(messages, ai_message_obj)
```

## Summary
基础部分已经完成, 可以看到这个 coding agent 已经可以读取文件 + 修改文件 + 列出目录下所有文件, 但仍有大量的改进空间:
- agent 没有进行多步工具调用和消息循环, 如果要对比5个文件, 可能只调用两个就结束了
- agent 没有规划能力, 这一般是 AI 编程助手的默认能力
- 没有网络搜索能力
- 使用 Rust 重写

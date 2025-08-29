+++
date = '2025-08-28T8:00:00+08:00'
draft = false
title = 'Prompt Organization'
tags = ['Prompt', 'LLMs']
+++

这篇文章旨在介绍 Python 中常用的提示词组织方式

###  f-string
使用 f 字符串填充变量得到提示词
```Python
def get_prompt(query: str) -> list[dict]:
    SYSTEM_PROMPT = f"""...
...
多行提示词, 也可以填充变量
"""
    USER_PROMPT = f"""INPUT:
{query}
....
"""
    return [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": USER_PROMPT},
    ]
```
这种方法实现简单, 速度快, 但是:
1. 多行字符串由于填充变量的需要, 需写在函数内, 导致代码格式混乱
2. 通过代码构造提示词, 任何修改都需要修改代码, 扩展性差


### string.Template
使用 Python 元素字符串模板
```Python
SYSTEM_PROMPT = string.Template("""你是一名$role
多行提示词...
""")

USER_PROMPT = string.Template("""INPUT:
$query
""")

def get_prompt(role: str, query: str) -> list[dict]:
    system_prompt = SYSTEM_PROMPT.subtitute(role="助手")
    user_prompt = USER_PROMPT.subtitute(query="问题...")
    return [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_prompt},
    ]
```
使用模板字符串, 模板则不必写在函数内, 且模板字符串可以选择替换部分变量, 使用 `.safe_substitute()`方法传入一个字典, 例如 `{"query": "问题..."}`, 对没有传入的变量解析为 `$var`  
对比 f-string, 模板字符串更加灵活, 且可以只传入部分值


### Jinja
Jinja 是一个现代的设计者友好的, 仿照 Django 模板的 Python 模板语言. 它速度快, 被广泛使用, 并且提供了可选的沙箱模板执行环境保证安全:  
例如下面这个 `.j2` 文件内容, 构造了一个用于少样本提示的模板
```Jinja
{% if examples %}
{% for example in examples %}
INPUT:
{{ example.input }}

OUPUT:
{{ example.output }}

{% endfor %}
{% endif %}
INPUT:
{{ user_input }}
```
导入该模板文件代码如下:
```Python
from jinja2 import Environment, PackageLoader # 根据需要不同也可以使用 FileSystemLoader

env = Environment(
    loader=PackageLoader("app.module.prompt", "template"),
    trim_blocks=True, # 移除 {% ... %} 块前后的多余空白
    lstrip_blocks=True, # 移除行首 {% ... %} 块前的空白
)

def get_prompt(user_input: str) -> list[dict]:
    system_template = env.get_template("system_template.j2")
    user_template = env.get_template("user_template.j2")

    system_data = {"var": val, ...}
    user_data = {
        "examples": [
            {"input": "示例输入1", "output": "示例输出1"}, # 具体样例也可以通过函数传入
            {"input": "示例输入2", "output": "示例输出2"},
        ],
        "user_input": user_input",
    }

    messages = [
        {"role": "system", "content": system_template.render(system_data)},
        {"role": "user", "content": user_template.render(user_prompt)},
    ]

    return messages
```
使用 Jinja 模板文件的好处是:
1. 方便组织提示词文件, 例如这里是将提示词文件放在 `ProjectRoot/app/module/prompt` 里面, 模板文件放在 `prompt/template` 里面, 在提示词文件中导入模板文件十分方便, 文件组织清晰, 代码可读性高, 且方便扩展
2. 提示词灵活性更好, 对比 string.Template, Jinja 模板不仅可以填充变量, 还可以在模板中插入循环和条件判断等语法, 使得代码中只需提供一个字典格式的数据即可, 无需在代码里拼凑提示词, 也方便和 RAG 系统结合使用

虽然 Jinja 对比 string.Template 性能上要差一些, 但是 LLM 应用真正花时间的地方是模型的推理部分, 相比之下提示词渲染的时间几乎可以忽略不计. 如果提示词非常多, Jinja 还提供了异步渲染功能, 可以结合异步框架进一步提升性能.

### Wrapping Up
上面就是近期使用的一些构造提示词的方法, 分别是 `f-string`、`string.Template` 和 `Jinja`.  
当然也有像 `langchain_core.prompts.prompt.PromptTemplate` 这样专用框架提供的提示词模板功能, 但是为了支持 LangChain LCEL 语法等原因, 导致其类型设计十分抽象, 且 LangChain 对新模型和新功能的支持比较缓慢, 加上版本不稳定, 接口经常变动, 故没有考虑使用 LangChain 框架提供的功能.(实际上, langchain 也支持使用 Jinja 模板)  
总之, 上面介绍的提示词构造方法各有优劣, 应该根据你项目的复杂度, 自行选择合适的提示词构造方式.

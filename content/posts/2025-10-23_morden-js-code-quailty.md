+++
date = '2025-10-23T8:00:00+08:00'
draft = false   
title = 'Code Quailty'
tags = ['Javascript']
+++

### 3.1 Debugging in the browser

Debugging is the process of finding and fixing errors with a script.
All morden browsers and most other environments support debugging tools - a special UI in developer tools that makes debugging much easier.
It also allows to trace the code step by step to see what exactly is going on.

#### The "Sources" panel

The Sources panel has 3 parts:

1. The **File Navigator** pane lists HTML, Javascript, CSS and other files, including images that are attatched to the page. Chrome extensions may appera here too.
2. The **Code Editor** pane shows the source code.
3. The **Javascript Debugging** pane is for debugging, we'll explore it soon.

#### Console

按 `Esc` 可以打开控制台，在其中可以输入命令，按回车执行。

#### Breakpoints

点击代码行号部分添加检查点，可以用于调试代码，暂停代码的执行。

也可以使用 `debuugger` 命令来暂停代码

```Javascript
function hello(name) {
    let prase = 'Hello, ${name}!';

    debugger;

    say(prase);
}
```

#### Pause and look around

代码框右侧有 3 个部分：

1. **`Watch` - shows current values for any expressions.**  
   监控表达式，该部分实时显示某个表达式的值

2. **`Call Stack` - shows the nested calls chain**  
   调用栈，显示当前执行位置的 “函数调用轨迹”

3. **`Scope` - current variables**  
   作用域，展示当前代码可访问的所有变量的地方
   - Local: 当前函数内部局部变量
   - Closure: 函数外层作用域闭包变量
   - Global: 全局作用域变量
   - this: 当前上下文绑定的对象

#### Logging

要在终端输出内容，使用 `console.log` 函数。
例如输出 `0` 到 `4`：

```Javascript
for (let i = 0; i < 5; i++) {
    console.log("value", i)
}
```

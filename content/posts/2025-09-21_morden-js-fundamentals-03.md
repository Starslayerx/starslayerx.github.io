+++
date = '2025-09-21T8:00:00+08:00'
draft = false
title = 'Morden Javascript Tutorial Chapter 2 - Fundamentals: 11~18'
tags = ['Javascript']
+++

## 2.11 Logical operators
There are four logical operators in JavaScript: `||` (ORA), `&&` (AND), `!` (NOT), `??` (Nullish Coalescing 空值合并).

### || OR
```JavaScript
result = a || b
```
There are four logical combinations:
```JavaScript
alter( true || true );   // true
alert( false || true );  // true
alert( true || false );  // true
alert( false || false ); // false
```
If an operand is not a boolean it's converted to be a boolean for the evaluation.
```JavaScript
if (1 || 0) { // works like (true || false)
    alter('truthy!');
}
```

#### OR "||" finds the first truthy value
```JavaScript
result = value1 || value2 || value3
```
The OR `||` operator does the following:
- Evaluates operands from left to righit
- For each operand, converts it to boolean. if the result is `true`, stops and returns the original value of that operand
- If all operands are false, returns the last operand.

```JavaScript
alter( 1 || 0 ); // 1
alter( null || 1 ); // 1
alter( null || 0 || 1 ); // 1
alter( undefined || null || 0 ); // 0 (all falsy)
```

If an operand is not a boolean, it's converted to a boolean for the evaluation.
For instance, the number `1` is treated as `true`, the `0` numebr as `false`.
```JavaScript
if (1 || 0) { // like if (true || false)
    alert( 'truthy!' );
}
```

This leads something interesting usage compared to a "pure, classical, boolean-only OR".

1. **Getting the first truthy value from a list of variables or exporessions**  
    For instance, we have `firstName`, `lastName`, and `nickName` variabels, all optional.
    Let's use OR `||` to choose the one that has the data and show it:
    ```JavaScript
    let firstName = ""
    let lastName = ""
    let nickName = ""
    alert(firstName || lastName || nickName || "Anonymous"); // SuperCode
    ```
    If all variables set `false`, "Annoymous" will show up.

2. **Short-cricuit evaluation**  
    It means that `||` processes its arguments untill the first truthy value is reached, and then the value is returned immediately, without even touching the other argument.
    ```JavaScript
    false || alert("not printed")
    true || alert("printed")
    ```
    Sometimes, people use this feature to execute commands only if the condition on the left part is falsy.

#### && (AND)
The AND operator is represented with two ampersands `&&`.
```JavaScript
result = a && b
```

#### AND "&&" finds the first falsy value
Given mulitple AND'ed values:
```JavaScript
result1 && result2 && result3
```
AND returns the first falsy value or the last value if none were found.  
Example:
```JavaScript
// if the first operand is truthy, returns the second operand:
alert( 1 && 0 ); // 0
alert( 1 && 5 ); // 5

// if the first operand is falsy, returns the it. The second operand is ignored.
alert( null && 5 ); // null
alert( 0 && "no matter what" ); // 0
```

#### ! (NOT)
The boolean NOT operator is represented with an exclamation sign `!`.
The syntax is pretty simple:
```JavaScript
result = !value;
```
The exclamation operator a single argument and does the following:  
1. Converts to operand to boolean type: `true/false`.
2. Returns the inverse value.

```JavaScript
alert( !true ); // false
alert( !0 ); // true
```
A double NOT `!!` is sometimes used for converting a value to boolean type:
```JavaScript
alert( !!"non-empty string" ); // true
alert( !!null ); // false
```


> Tasks

**What's the result of OR'ed alerts?**

```JavaScript
alert( alert(1) || 2 || alert(3) );
```
Answer: `1` and `2`  
1. The OR `||` evaluates `alter(1)`. That's shows message 1 and returns 'undefined'.
2. The `alter` returns `undefined`, so OR goes into the second operand.
3. The `2` is truthy, so the execution is halted (停止), `2` is returned and then shown by the outer alert.

---

**What is the result of AND'ed alerts?**

```JavaScript
alert( alert(1) && alert(2) );
```
Answer: `1` and `undefined`  

---

**Check the range between**

Write an `if` condition to check that `age` is between `14` and `90` inclusively.
```JavaScript
if (age >= 14 && age <= 90) {
    alert(`Age is ${age}. Is between 14 and 90.`);
}
```

---

**Check the range outside**

Write and `if` condition to check that `age` is NOT between `14` and `90` inclusively.
Create two variants (变体): the first one using NOT `!`, the second one - withoud it.
```JavaScript
if (!(age >= 14 && age<=90)) {
    alert(`Age is ${age}. Not between 14 and 90.`);
}

if (age < 14 || age > 90) {
    alert(`Age is ${age}. Not between 14 and 90.`);
}
```

---

**A question about "if"**

Which of these alerts are going to execute?
What will the results of the expressions be inside if(...)?

```JavaScript
if (-1 || 0) alert( 'first' );  // runs 
if (-1 && 0) alert( 'second' ); // doesn't run

if (null || -1 && 1) alert( 'third' );
// ->  null || (-1 && -1)
// ->  null || -1
// -> -1
```
Operator && has higher precedence (优先级) that ||.

---

**Check the login**

Write the code which asks for a login with `prompt`.

If the visitor enters `"Admin"`, the `prompt` for a password, if the input is an empty line or `Esc` – show “Canceled”, if it’s another string – then show “I don’t know you”.

The password is checked as follows:

- If it equals “TheMaster”, then show “Welcome!”,
- Another string – show “Wrong password”,
- For an empty string or canceled input, show “Canceled”

```JavaScript
let user = prompt("请输入用户名：", "");

if (user === "" || user === null) {
    alter( "Canceled" );
} else if (user === "Admin") {
    let pass = prompt("请输入密码：", "");
    if (pass == "" || pass == null) {
        alert( "Canceled" );
    } else if (pass == "TheMaster") {
        alert( "Welcome!" );
    } else {
        alert( "Wrong password" );
    }
} else {
    alter( "I don't know you" );
}
```
注意点：  
- `prompt(text, defaultText)`: JavaScript 中用于获取用户输入的浏览器内置方法，它会显示一个对话框，包含提示信息、输入框和确定/取消按钮。`text` 为话框中显示的提示文本， `defaultText` 为输入框中的默认值。
- 从 `prompt()` 获取的输入可能为空字符，也可能为 `null`。
- 等号使用严格等于 `===` 而不是 `==`。


## 2.12 Nullish coalescing operator '??'
The nullish coalescing operator (空值合并运算符) is written as two question marks `??`.

It's similar to `||` but only consider `null` and `undefined` as falsy. (`||` consider `0`, `""`, `false`, `null` and `undefined` as falsy.)

The result of `a ?? b` is:
- if `a` is defined, then `a`
- if `a` is not defined, then `b`

In other words, `??` returns the first argument if it’s not `null/undefined`.

We can rewrite `result = a ?? b` using operators we alreadly know, like this:
```JavaScript
result = (a !== null && a !== '') ? a : b;
```

The common use for `??` is to provide default value.
```JavaScript
let user;
alert(user ?? "Anonymous"); // "Annoymous"

user = "John";
alert(user ?? "Anonymous"); // "John"
```

We can alos use a sequence of `??` to select the first value from a list that isn't `null/undefined`.
```JavaScript
let firstName = null;
let lastName = null;
let nickName = "Supercoder";
alert(firstName ?? lastName ?? nickName ?? "Anonymous"); // Supercoder
```

#### Precedence
The precedence of `??` and `||` operator is the same, they both equal `3` in the [MDN table](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence#Table) (higher than `=` and `?`, but lower than `+` and `*`).

#### Using ?? with && or ||
Due to safety reasons, JavaScript forbids `??` together with `&&` and `||` operators, unless the precedence is explicitly specificed with parentheses (括号).

The code below triggers a syntax error:
```JavaScript
let x = 1 && 2 ?? 3; // Syntax error
```
Use explicit parentheses to work around it:
```JavaScript
let x = (1 && 2) ?? 3; // Works
alert(x); // 2
```

## 2.13 Loops: while and for

#### The "while" loop
The `while` loop has the following syntax:
```JavaScript
while (condition) {
    // code: loop-body
}
```

#### The "do...while" loop
```JavaScript
do {
    // lop body
} while (condition);
```

#### The "for" loop
```JavaScript
for (let i = 0; i < 3; i++) {
    alert(i); // 0, 1, 2
}
```

#### Breaking the loop
But we can force the exit at any time using the special `break` directive.
```JavaScript
let sum = 0;
while (true) {
    let value = +prompt("Enter a number", ""); // + 号将 prompt() 获取的字符串转化为数字
    if (!value) break; // (*) 非数字
    sum += value;
}

alert("sum" + sum);
```

#### Continue to the next iteration
The continue directive (指令) is a “lighter version” of break. It doesn’t stop the whole loop. Instead, it stops the current iteration and forces the loop to start a new one.
```JavaScript
for (let i = 0; i < 10; i++) {
    if (i % 2 === 0) continue; // skip alert enum numbers
    alert(i); // 1, then 3, 5, 7, 9
}
```

- Note that syntax constructs that are not expressions cannot be used with the ternary operator `?`. In particular, directives such as `break/continue` aren’t allowed there.
    ```JavaScript
    if (i > 5) {
        alert(i);
    } else {
        continue;
    }
    ```
    can't be write as
    ```JavaScript
    (i > 5) ? alert(i) : continue; // Syntax error: continue isn't allowed here
    ```

#### Label for break/continue
A label is an identifier with a colon before a loop:
```JavaScript
labelName: for (...) {
    ...
}
```
A label can break mulitple loop, and `break/input` would break inner loop:
```JavaScript
outer: for (let i = 0; i < 3; i++) {
    for (let j = 0; j < 3: j++) {
        let input = prompt(`Value at corrds (${i}, ${j})`, '');
        if (!input) break outer; // (*)
    }
}
alter("Done!");
```
In the code above, `break outer` looks upwards for the label named `outer` and breaks out of that loop.
So the control goes straight from `(*)` to `alert('Done!')`.

The `continue` directive can also be used with a label.
In this case, code execution jumps to the next iteration of the labeled loop.


## 2.14 The "switch" statement
A `switch` statement can replace multiple `if` checks.
It gives a more descriptive way to compare a value with multiple variants.

#### The syntax
The `switch` has one or more `case` blocks and an optional default.
```JavaScript
switch (x) {
    case 'value1': 
        ...
        [break]
    case 'value2':
        ...
        [break]
    default:
        ...
        [break]
}
```

#### An example
An example of `switch`
```JavaScript
let a = 2 + 2;

switch (a) {
    case 3:
        alert( 'Too small' );
        break;
    case 4:
        alert ( 'Exactly' );
        break;
    case 5:
        alert ( 'Too big' );
        break;
    default:
        alert("I don't know such values");
}
```
Here the `switch` starts to compare `a` from the first `case` untill `case 4` and then breaks.

If there is no `break` then the execution continues with the next `case` without any checks.
```JavaScript
let a = 2 + 2;

switch (a) {
    case 3:
        alert( 'Too small' );
    case 4:
        alert( 'Exactly!' );
    case 5:
        alert( 'Too big' );
    default:
        alert( "I don't know such valeus" );
}
```
In the example above we’ll see sequential execution of three `alert`s:
```JavaScript
alert( 'Exactly!' );
alert( 'Too big' );
alert( "I don't know such values" );
```

#### Grouping of "case"
Serval variants of `case` which share the same code can be grouped.
For example, we want the same code for `case 3` and `case 5`:
```JavaScript
let a = 3;
switch (a) {
    case 4:
        alert('Right!');
        break;

    case 3:
    case 5:
        alert('Wrong!');
        break;

    default:
        alert('The result is strange. Really.');
}
```
Now both `3` and `5` show the same message.

#### Type matters
The equality check is always strict.
The values must be of the same type to match.

For example:
```JavaScript
let arg = prompt('Enter a value?');

switch (arg) {
    case '0':
    case '1':
        alert( 'One or zero' );
        break;

    case '2':
        alert( 'Two' );
        break;

    case 3: // '3' will not execute this block
        alert( 'Never executes!' );
        break;

    default:
        alert( 'An unkown value' );
}
```

## 2.15 Functions

#### Function Declaration
To create a function, use `function` declaration:
```JavaScript
function name(parameter1, paprameter2, ... parameterN) {
    // body
}
```

#### Local variables
A variable declared inside a function is only visible inside that function.
```JavaScript
function showMessage() {
  let message = "Hello, I'm JavaScript!"; // local variable

  alert( message );
}

showMessage(); // Hello, I'm JavaScript!

alert( message ); // <-- Error! The variable is local to the function
```

#### Outer variables
A function can access an outer variable as well.
```JavaScript
let userName = 'John';

function showMessage() {
  let message = 'Hello, ' + userName;
  alert(message);
}

showMessage(); // Hello, John
```

The function has full access to the outer variable. It can modify it as well.
```JavaScript
let userName = 'John';

function showMessage() {
  userName = "Bob"; // (1) changed the outer variable

  let message = 'Hello, ' + userName;
  alert(message);
}

alert( userName ); // John before the function call

showMessage();

alert( userName ); // Bob, the value was modified by the function
```

The outer variable is only used if there’s no local one.

If a same-named variable is declared inside the function then it shadows the outer one. For instance, in the code below the function uses the local `userName`. The outer one is ignored:
```JavaScript
let userName = 'John';

function showMessage() {
  let userName = "Bob"; // declare a local variable

  let message = 'Hello, ' + userName; // Bob
  alert(message);
}

// the function will create and use its own userName
showMessage();

alert( userName ); // John, unchanged, the function did not access the outer variable
```

#### Parameters
We can pass arbitrary data to functions using parameters.
In the example below, the function has two parameters: `from` and `text`.
```JavaScript
function showMessage(from, text) { // parameters: from, text
  alert(from + ': ' + text);
}

showMessage('Ann', 'Hello!'); // Ann: Hello! (*)
showMessage('Ann', "What's up?"); // Ann: What's up? (**)
```

#### Default Values
If a function is called, but an argument is not provided, then the corresponding value becomes `undefined`.

For example, the aforementioned function `showMessage(from, text)` can be called with a single argument.
```JavaScript
showMessage("Ann");
```
That’s not an error. Such a call would output `"*Ann*: undefined"`.
As the value for text isn’t passed, it becomes `undefined`, like this:
```JavaScript
showMessage("Ann", undefined); // Ann: no text given
```

We can specify the so-called “default” (to use if omitted) value for a parameter in the function declaration, using `=`:
```JavaScript
function showMessage(from, text = "no text given") {
  alert( from + ": " + text );
}

showMessage("Ann"); // Ann: no text given
showMessage("Ann", undefined); // Ann: no text given
```
Here `"no text given"` is a string, but it can be a more complex expression, which is only evaluated and assigned if the parameter is missing. So, this is also possible:
```JavaScript
function showMessage(from, text = anotherFunction()) {
  // anotherFunction() only executed if no text given
  // its result becomes the value of text
}
```

- **Alternative default parameters**  
    Sometimes it makes sense to assign default values for parameters at a later stage after the function declaration.  
    We can check if the parameter is passed during the function execution, by comparing it with `undefined`:
    ```JavaScript
    function showMessage(text) {
      if (text === undefined) { // if the parameter is missing
        text = 'empty message';
      }

      alert(text);
    }

    showMessage(); // empty message
    ```
    …Or we could use the || operator:
    ```JavaScript
    function showMessage(text) {
      // if text is undefined or otherwise falsy, set it to 'empty'
      text = text || 'empty';
      ...
    }
    ```
    Modern JavaScript engines support the nullish coalescing (合并) operator `??`, it’s better when most falsy values, such as `0`, should be considered “normal”:
    ```JavaScript
    function showCount(count) {
      // if count is undefined or null, show "unknown"
      alert(count ?? "unknown");
    }

    showCount(0); // 0
    showCount(null); // unknown
    showCount(); // unknown
    ```

#### Returning a value
A function can return a value back into the calling code as the result.
The simplest example would be a function that sums two values:

```JavaScript
function sum(a, b) {
  return a + b;
}

let result = sum(1, 2);
alert( result ); // 3
```

If a function does not return a value, it is the same as if it returns undefined:
```JavaScript
function doNothing() { /* empty */ }
alert( doNothing() === undefined ); // true
```
An empty return is also the same as return undefined:
```JavaScript
function doNothing() {
  return;
}

alert( doNothing() === undefined ); // true
```

#### Naming a function
Functions are actions. So their name is usually a verb. 
It is a widespread practice to start a function with a verbal prefix which vaguely describes the action.

Function starting with…

- `"get…"` – return a value,
- `"calc…"` – calculate something,
- `"create…"` – create something,
- `"check…"` – check something and return a boolean, etc.

```JavaScript
showMessage(..)     // shows a message
getAge(..)          // returns the age (gets it somehow)
calcSum(..)         // calculates a sum and returns the result
createForm(..)      // creates a form (and usually returns it)
checkPermission(..) // checks a permission, returns true/false
```

- **One function – one action**  
    A function should do exactly what is suggested by its name, no more.  
    Two independent actions usually deserve two functions, even if they are usually called together (in that case we can make a 3rd function that calls those two).

- **Ultrashort function names**  
    Functions that are used very often sometimes have ultrashort names.  
    For example, the jQuery framework defines a function with `$`. The Lodash library has its core function named `_`.  
    These are exceptions. Generally function names should be concise and descriptive.

#### Function == Comments
Functions should be short and do exactly one thing.
If that thing is big, maybe it’s worth it to split the function into a few smaller functions.
Sometimes following this rule may not be that easy, but it’s definitely a good thing.

A separate function is not only easier to test and debug – its very existence is a great comment!

For instance, compare the two functions `showPrimes(n)` below.
Each one outputs prime numbers up to `n`.

The first variant uses a label:
```JavaScript
function showPrimes(n) {
  nextPrime: for (let i = 2; i < n; i++) {

    for (let j = 2; j < i; j++) {
      if (i % j == 0) continue nextPrime;
    }

    alert( i ); // a prime
  }
}
```
The second variant uses an additional function `isPrime(n)` to test for primality:
```JavaScript
function showPrimes(n) {

  for (let i = 2; i < n; i++) {
    if (!isPrime(i)) continue;

    alert(i);  // a prime
  }
}

function isPrime(n) {
  for (let i = 2; i < n; i++) {
    if ( n % i == 0) return false;
  }
  return true;
}
```
The second variant is easier to understand.

> Tasks

**Rewrite the function using '?' or '||'**

```JavaScript
function checkAge(age) {
  if (age > 18) {
    return true;
  } else {
    return confirm('Did parents allow you?');
  }
}

```
Rewrite it, to perform the same, but without `if`, in a single line.  
Make two variants of `checkAge`:  
    1. Using a question mark operator `?`  
    2. Using OR `||`  
```JavaScript
function checkAge(age) {
    return age > 18 ? true : confirm('Did parents allow you?');
    return true === (age > 18) || confirm('Did parents allow you?');
}
```

## 2.16 Function expressions
*Function Declaration*
```JavaScript
function sayHi() {
    alert( "Hello" );
}
```

*Function Expression*
```JavaScript
let sayHi = function() {
    alert( "Hello" );
}; // semicolon 分号
```

#### Function is a value
Let's reiterate (重申): no matter how the function is created, a function is a value.
Both examples above sotre a function in the `sayHi` variable.
```JavaScript
function sayHi() {
    alert( "Hello" );
}
alert( sayHi ); // shows the function code
```
The lastline won't run the code, because there are no parentheses after `sayHi`.

We can copy a function to another variable.
```JavaScript
function sayHi() {
    alert( "Hello" );
}
let func = sayHi;
func();  // "Hello"
sayHi(); // "Hello"
```

#### Callback functions
```JavaScript
function ask(question, yes, no) {
    if (confirm(question)) yes() // "确定" / "取消" 按钮对话框
    else no();
}

function showOk() {
    alert( "You agred." );
}

function showCancel() {
    alert( "You canceled the execution." );
}

ask("Don you agres?", showOk, showCancel);
```
**The arguments `showOk` and `showCancel` of `ask` are called callback functions or just callbacks.**

The idea is that we pass a function and except it to be "called back" later if necessary.
In this case, `showOk` becomes the callback for "yes" answer, and `showCancel` fo "no" answer.

We can use *Function Expressions* to write a equivalent shorter function:
```JavaScript
function ask(question, yes, no) {
    if (confirm(question)) yes()
    else no();
}

ask(
    "Do you agree?",
    function() { alert("You agreed."); },
    function() { alert("You didn't agree."); },
)
```

#### Function Expression vs Function Declaration
The key difference between Function Expression and Function Declaration:

- *Function Declaration*: a function, declared as a separate statement.
    ```JavaScript
    function sum(a, b) {
        return a + b;
    }
    ```

- *Function Expression*: a function, declaured inside an expression or inside anther syntax construct.
    ```JavaScript
    let sum = function(a, b) {
        return a + b;
    };
    ```

The more subtle difference is when a function is created by the JavaScript engine:

- **A Function Expression is created when the execution reaches it and is usable only from that moment.**

- **A Function Declaration can be called earlier than it is defined.**

**In strict mode, when a Function Declaration is within a code block, it's visible everywhere inside that block. But not outside of it.**
```JavaScript
let age = prompt("What's your age?", 18);

if (age < 18) {
    function welcome() {
        alert('Hello');
    }
} else {
    function welcome() {
        alert('Greetings!');
    }
}

welcome(); // Error: welcome is not defined
```
The correct approch would be to use a Function Expression and assign `weclome` to the variable that is declared outside of `if` and has the proper visibility.
```JavaScript
let age = prompt("What is your age?", 18);
let welcome;

if (age < 18) {
    welcome = function() {
        alert('Hello');
    };
} else {
    welcome = function() {
        alert('Greetings');
    }
}

welcome(); // ok now
```
We can simplify it even further using an question mark operator `?`:
```JavaScript
let age = prompt("What's your age?", 18);

let welcome = age < 18 ? 
    function() { alert('Hello'); } :
    function() { alert('Greetings'); };
welcome(); // ok
```

## 2.17 Arrow functions, the basics
"arrow functions" looks like this:
```JavaScript
let func = (arg1, arg2, ..., argN) => expression;
```
let's create some concrete examples:
```JavaScript
let sum = (a, b) => a + b;
alert( sum(1, 2) ); // 3

// one arugemnt
let double = (a) => a * 2;
alert( double(1) ); // 2

// no argument
let sayHi = () => alert("Hello!");
alert( sayHi() );
```

Arrow functions can be used in the same way as Function Expressions.
```JavaScript
let age = prompt("What's your age?", 18);

let welcome = (age < 18) ?
    () => alert('Hello') :
    () => alert('Greetings');

welcome()
```

#### Multiline arrow functions
Sometime we need a more complex function with multiple expressions.
In that case, we can enclose (包含) them in curly braces (大括号).
```JavaScript
let sum = (a, b) => {
    let result = a + b;
    return result; // with curly braces, need an explicit "return"
}
alert(1, 2); // 3
```

---

**Rewrite with arrow functions**

Replace Function Expressions with arrow functions in the code below:
```JavaScript
function ask(question, yes, no) {
  if (confirm(question)) yes();
  else no();
}

ask(
  "Do you agree?",
  function() { alert("You agreed."); },
  function() { alert("You canceled the execution."); }
);
```

Note that there is an semicolon after a function call (two parentheses).
```JavaScript
function ask(question, yes, no) {
    if (confirm(question)) yes(); // 注意分号
    else no();
}

ask(
    "Do you agree?",
    () => alert("You agreed."),
    () => alert("You canceled the execution.")
);
```


## 2.18 JavaScript specials

#### Code Structure
Statements are delimited (分隔) with semicolon (分号):
```JavaScript
alert('Hello'); alert('World');
```
Usually a line-break is also treated as a delimiter (分隔符), so that would also work.
```JavaScript
alter('Hello')
alter('World')
```
It's called "automatic semicolon insertion". Sometimes it doesn't work, for instance:
```JavaScript
alert("There will be an error after this message")

[1, 2].forEach(alert)
```

Semicolon are not required after code blocks `{...}` and syntax constructs with them like loops:
```JavaScript
function f() {
} // no semicolon needed

for(;;) {
} // no semicolon needed
```

#### Strict mode
To fully enable all features of morden JavaScript, use start script `"use strict"`
```JavaScript
"use strict"

...
```
The directive (指令) must at the top of the script or at the beganing of a function body.

#### Variables
Can be declared using:
- `let`
- `const`
- `var` (old style)

A variable name can include:
- letter and digits, but the first character may not be digit.
- Characters `$` and `_` are normal
- Non-latin aiphabets and hieroglyphs (象形文字) are also allowed, but commonly not used.

There are 8 data types:
- `number` for both floating-point and integer numbers.
- `bigint` for integer of arbitrary length.
- `string` fot strings
- `boolean` for logical values `true/false`
- `null` a type with single value, meaning "empty" or "doesn't not exist"
- `undefined` a type with single value, meaning "not assigned"
- `object` and `symbol` - for complex data structures and unique identifiers

The `typeof` operator returns the type for a value, with two execptions:
```JavaScript
typeof null == "object" // an error in the language
typeof function() {} == "function" // functions are treated speically
```

#### Interaction
Basic browser UI functions

- `prompt(question, [default])`  
    Ask a `question`, and return either the visitor entered or `null` if they click "cancel".

- `confirm(question)`  
    Ask a `question`, and suggest to choose Ok and Cancel. The choice is returned as `true/false`.

- `alert(message)`  
    Output a `message`.

For instance:
```JavaScript
let userName = prompt("Your name?", "Alice");
let isTeaWanted = confirm("Do u want tea?");

alert( "Visitor: " + userName );
alert( "Tea wantd: " + isTeaWanted );
```

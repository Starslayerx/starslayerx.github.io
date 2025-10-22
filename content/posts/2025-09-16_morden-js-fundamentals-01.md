+++
date = '2025-09-16T8:00:00+08:00'
draft = false
title = 'Morden Javascript Tutorial Chapter 2 - Fundamentals: 01~05'
tags = ['JavaScript']
+++

## 2.1 Hellow Wrold
Firtly, let's see how to attach a script to a webpage.
For server-side environments (like Node.js), you can execute this script with a command like `node my.js`

### The "script" tag
JavaScript programs can be inserted almost anywhere into an HTML docuemnt using the `<script>` tag

```html
<!DOCTYPE HTML>
<html>
<body>
    <p> Before this script...</p>
    <script>
        alert( 'Hello, World!' );
    </script>
    <p>...After the script...</p>
</body>
</html>
```

The `<script>` tag contains JavaScript code which is automatically executed when the browser process the tag.

### Modern markup
The `<script>` tag has a few attributes that are rarely used nowadays but can still be found in old code:

- The `type` attribute: `<script type=...>`

The old HTML standrad, HTML 4, required a script to have a `type`.
Usually it was `type="text/javascript"`. It's not required anymore.
Also, the modern HTML standrad totally changed the meaning of this attribute.
Now, it can be used for JavaScript modules.

- The `language` attribute: `<script language=...>`

This attribute was meant to show the language of the script.
This attrubute no longer makes sense because JavaScript is the default language.
There is no need to use it.

- Comments before and after scripts

In really ancient books and guides, you may find comments inside `<script>` tage, like this:
```html
<script type="text/javascript"> <!--
    ...
//--></script>
```
This trick isn't used in modern JavaScript.
This comments hide JavaScript code from old browser that didn't know how to process the `<script>` tag.
This kind of comment can help you identify really old code.

### External scripts
If we have a lot of JavaScript code, web can put it into a separate file.
Script files are attached to HTML with the `src` attribute:
```html
<script src="/path/to/script.js"></script>
```
We can use a url as well.
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/lodash.js/4.17.11/lodash.js"></script>
```
To attach serval scripts, use multiple tags:
```html
<script src="/js/script1.js"></script>
<script src="/js/script2.js"></script>
```

> if `src` is set, the script content is ignored.

A single `<script>` tag can't have both the `src` attrubute and code inside.
```html
<script src="file.js">
  alert(1); // the content is ignored, because src is set
</script>
```

We must choose either an external `<script src="...">` or a regulr `<script>` with code.
```html
<script src="file.js"></script>
<script>
  alert(1);
</script>
```

## 2.2 Code Structure

### Statements
Statements are syntax constructs and commands that perform actions.  
We can have as many statements in our code as we want. Statemets can be seperated with a semicolon.
```JavaScript
alert('Hello');
alert('World');
```

### Semicolons
A Semicolon may be omitted in most cases when a line break exists.
```JavaScript
alert('Hello')
alert('World')
```

Here, JavaScript interprets the line break as an "implicit" semicolon.
This is called an [automatic semicolon insertion](https://tc39.github.io/ecma262/#sec-automatic-semicolon-insertion)

> An example of an error

```JavaScript
alert("Hello");
[1, 2].forEach(alert);
```
Now remove the semicolon after the `alert`
```JavaScript
alert("Hello")
[1, 2].forEach(alert);
```
If we run this code, only the first `Hello` shows.
That's because JavaScript does not assume a semicolor before square brackets `[...]`.
So, the code in the last example is threated as a single statement.

Here is how the engine see it:
```JavaScript
alert("Hello")[1, 2].forEach(alert);
```
It is possible to leave out semicolons most of the time.
But it’s safer – especially for a beginner – to use them.

### Comments
Comments can be put into any place of a script.
They don't affect its execution because the engine simply ignores them.

One-line comments start with two forward slash characters `//`.
```JavaScript
// This comment occupies a line of its own
alert('Hello');
alert('World'); // This comment follows the statement
```

Multiple comments start with a forward slash and an asterisk `/*` and end with an asterisk adn a forward slash `*/`.

```JavaScript
/* An example with two messages.
This is a multiline comment.
*/
alert('Hello');
alert('World');
```

> Nested comments are not supported!

There may not be `/*...*/` inside another `/*...*/`
```JavaScript
/*
  /* nested comment ?!? */
*/
alert( 'World' );
```

## 2.3 The Morden Mode, "use strict"
For a long time, JavaScript evolved without compatibility issues.
New features were add to the language while old functionality didn't change.

That had the benefit of never breaking existing code.
But the downside was that any mistake or an imperfect descision made by JavaScript's creators got stuck in the language forever.

This was the case until 2009 when ECMAScript 5 appered. It add new features to the language and modified some of the existing ones.
To keep the old code working, most such modifications are off by default.
You need to explicitly enable them with a special directive: `"use strict"`

### "use strict"
The directive looks like a string: `"use strict"` or `'use strict'`.
When it is located at the top or a script, the whole script works the "modern" way.

```JavaScript
"use strict"

// This code works the morden way
```

`"use strict"` can be put at the beginning of a function.
Doing that enables strict mode in that function only.
But usually people use it for the whole script.

> Ensure that "use strict" is at the top.  
> And there's no way to cancel `use strict`  
> There is no directive like `"no use strict"` that reverts the engine to old behavior. Once we enter strict mode, there’s no going back.


### Browser console
When you use a [developer console](https://javascript.info/devtools) to run code, please note that it doesn’t `use strict` by default.  
Sometimes, when `use strict` makes a difference, you’ll get incorrect results.  
```JavaScript
'use strict'; // <Shift+Enter for a newline>
//  ...your code
<Enter to run>
```
It works in most browsers, namely Firefox and Chrome.
If it doesn’t, e.g. in an old browser, there’s an ugly, but reliable way to ensure use strict.
Put it inside this kind of wrapper:
```JavaScript
(function() {
  'use strict';
  // ...your code here...
})()
```

### Should we "use strict"?
The question may sound obvious, but it’s not so.  
One could recommend to start scripts with `"use strict"`… But you know what’s cool?  

Modern JavaScript suuports "classes" and "modules" - advanced language structures, that enable `use strict` automatically.
So we don't need to add the `"use strict"` directive, if we use them.  
Later, when your code is all in classes and modules, you may omit it.



## 2.4 Variables
To create a variable in JavaScript, use the `let` keyward.
```JavaScript
let messages;
```

Now, we can put some data into it by using assignment operator `=`
```JavaScript
let message;
message = 'Hello';
```
To be concise, we can combine the variable declaration and assignment into a single line:
```JavaScript
let message = 'Hello';
```
We can also declare multiple variables in one line:
```JavaScript
let user = 'John', age = 25, message = 'Hello';
```
The multiline variant is a bit longer, but easier to read:
```JavaScript
let user = 'John';
let age = 25;
let message = 'Hello';
```
Some people also define multiple variables in this multiline style:
```JavaScript
let user = 'John'
  , age = 25
  , message = 'Hello';
```
Technically, all these variants do the same thing. So, it’s a matter of personal taste and aesthetics.

> `var` instead of `let`

In older scripts, you may also find another keyword: `var` instead of `let`:
```JavaScript
var message = 'Hello';
```
The `var` keyword is almost the same as `let`.
It also declares a variable but in a slight diffent, "old-school" way.

There are subtle differences between `let` and `var`, but they do not matter to us yet.

> Declaring twice triggers an error

A variable should be declared only once.  
A repeated declaration of the same variable is an error:
```JavaScript
let message = "This";

// repeated 'let' leads to an error
let message = "That"; // SyntaxError: 'message' has already been declared
```

### Variable naming
There are two limitations on variable names in JavaScript:  
1. The name must contain only letters, digits, or symbols `$` and `-`.
2. The first character must not be a digit.

```JavaScript
let userName;
let test123;
```

When the name contains multiple words, [camelCase](https://en.wikipedia.org/wiki/CamelCase) is commonly used.
That is: words go one after another, each word execpt first starting with a captital letter: `myVeryLongName`.

What's intersting - the doller sign `$` and the underscore `_` can also be used in names.
They are regular symbols, just like letters, without any special meaning.

```JavaScript
let $ = 1; // declared a variable with the name "$"
let _ = 2; // and now a variable with the name "_"

alert($ + _); // 3
```

Examples of incorrect variable names:
```JavaScript
let 1a; // cannot start with a digit
let my-name; // hyphens '-' aren't allowed in the name
```

> Case Matters

Variables named `apple` and `APPLE` are two different variables.


> Non-Latin letters are allowed, but not recommended

It is possible to use any language, including Cyrillic letters, Chinese logograms and so on, like this:

```JavaScript
let имя = '...';
let 我 = '...';
```

> Reserved names

There is a [list of reserved words](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Lexical_grammar#Keywords), which cannot be used as variable names because they are used by the language itself.  
For example: `let`, `class`, `return`, and `function` are reserved.  
The code blew gives a syntax error:
```JavaScript
let let = 5; // can't name a variable "let", error!
let return = 5; // also can't name it "return", error!
```

> An assignment without `use strict`

Normally, we need to define a variable before using it.
But in the old times, it was technically possible to create a variable by a mere assignment of the value without using `let`.
This still works now if we don't put `use strict` in our script to maintain compatibility with old scripts.
```JavaScript
// note: no "use strict" in this example
num = 5; // the variable "num" is created if it didn't exist
alert(num); // 5
```
This is a bad practice and would cause an error in strict mode:
```JavaScript
"use strict";
num = 5; // error: num is not defined
```

### Constants
To declare a constant variable, use `const` instead of `let`:
```JavaScript
const myBirthday = '18.04.1982';
```

Variables declared using `const` are called "constants"; They cannot be reassigned.
An attempt to do so would cause an error:
```JavaScript
const myBirthday = '18.04.1982';
myBirthday = '01.01.2001'; // error: can't reassign the constatn!
```
When a programmer is sure that a variable will never change, they can declare it with `const` to guarantee and communicate that fact to everyone.


### Uppercase constants
There is a widespred pratctice to use constants as aliases for difficult-to-remember values that are known before execution.  
Such constants are named using capital letters and underscores.  
For instance, let's make constants for colors in so-called "web" (hexadecimal) format:
```JavaScript
const COLOR_RED = "#F00";
const COLOR_GREEN = "#0F0";
const COLOR_BLUE =  "#00F";
const COLOR_ORANGE + "#FF7F00";

// ...when we need to pick a color
let color = COLOR_ORANGE;
alert(color);
```
Benefits:  
- `COlOR_ORANGE` is much easier to understand that "#FF7F00".
- It is much easier to mistype "#FF7F00" than `COLOR_ORANGE`.
- When reading the code, `COLOR_ORANGE` is much more meaningful than `#FF7F00`.

This is when should we use capitals for a constant:  
Being a "constant" just means that a variable's value never changes.
But some constants are known before execution and some constants are calculated in run-time,
during the execution, but do not change after their initial assignment.

For instance:  
```JavaScript
const pageLoadTime = /* time taken by a webpage to load */;
```
The value of `pageLoadTime` is not known before the page load, so it's named normally.
But it's still a constant because it doesn't change after the assignment.

In other words, capital-named constants are only used as aliases for "hard-coded" values.

### Name things right
Talking about variables, there's one more extremely important thing.

A variable name should have a clean, obvious meaning, describing the data that is stores.

Variable naming is one of the most important and complex skills in programming.
A glance at variable names can reveal which cdoe was written by a beginner versus an experienced developer.

In a real project, most of the time is spent modifying and extending an existing code base rather than writing something completely separate from scratch.
When we return to some code after doing something else for a while, it's much easier to find information that is well-labelled.
Or, in other words, when the variables have good names.

Please spend time thinking about the right name for a variable before declaring it.
Doing so will repay you handsomely.

Some good-to-follow rules are:  
- Use human-readable names like `userName` or `shoppingCart`.
- Stay away from abbreviations or short names like `a`, `b` and `c`, unless you know what you're doing.
- Make names maximallly descriptive and concise.
- Agree on terms within your team and your mind.

> Reuse or create?

There are some lazy programmers who, instead of declaring new variables, tend to reuse existing ones.

As a result, their variables are like boxes into which people different things without changding their stickers.
What's inside the box now? Who knows? We need to come closer and check.

Such programmers save a litte bit on variable declaration but lose ten time more on debugging.

An extra variable is good, not evil.

Morden JavaScript minifiers and browsers optimize code well enough, so it won't create performance issuse.
Using different variables for different values can even help engine optimize your code.


## 2.5 Data Types
A value in JavaScript always of a certain type.  
There are eight basic types in JavaScript.  
We can put any type in a variable.  
```JavaScript
// no error
let message = "hello";
message = 123456;
```
Programming languages that allow such things, such as JavaScript, are called "dynamically typed", meaning that there exist data types, but variables are not bound to any of them.

### Number
```JavaScript
let n = 123;
n = 12.345;
```
The *number* types represent both intger and floating point numbers.

There are many operations for numbers, e.g. multiplication `*`, division `/`, addition `+`, subtraction `-`, and so on.

Besides regular numbers, there are so-called "special numberic values" which also belong to this data type: `Infinity`, `-Infinity` and `NaN`.

- `Infinity` represents the mathematical infinity ∞. It is a special value that's greater than any number.  
  We can get it as a result of division by zero:
  ```JavaScript
  alert(0 / 0); // Infinity
  ```
  Or just reference it directly:
  ```JavaScript
  alert( Infinity ); // Infinity
  ```

- `NaN` represents a computational error. It is a result of an incorrect or an undefined mathematical operation, for instance: 
  ```JavaScript
  alert( "not a number" / 2 ); // NaN, such division is erroreous
  ```
  `NaN` is sticky. Any further mathematical operation on `NaN` returns `NaN`:
  ```JavaScript
  alert( NaN + 1 ); // NaN
  alert( NaN * 3 ); // NaN
  alert( "not a number" / 2 - 1 ); // NaN
  ```
  So, if there's a `NaN` somewhere in a mathematical expression, it propagates to the whole result.  
  There's one exception to that: `NaN ** 0` is `1`.

> Mathematical operations are safe

Doing maths is "safe" in JavaScript. We can do anything: divide by zero, threat non-numeric strings as numbers, etc.  
The script will never stop with a fatal error ("die"). At worst, we'll get `NaN` as the result.

### BigInt
In JavaScript, the "number" type cannot safely represent integer values larger than (`2^{53} - 1`), or less than `-(2^{53} - 1)` for negatives.

To be really precise, the "number" type can store larger integer (up to `1.7976931348623157 * 10^{308}`),
but outside of the safe integer range `±(2^{53}-1)` there'll be a precision error, because not all digits fit into the fixed 64-bit storage. So an "approximate" value may be stored.

For example, these two number are the same:
```JavaScript
console.log(9007199254740991 + 1); // 9007199254740992
console.log(9007199254740991 + 2); // 9007199254740992
```

So to say, all odd integers greater than `(2^{53}-1)` can’t be stored at all in the “number” type.

For most purposes `±(2^{53}-1)` range is quite enough, but sometimes we need the entire range of really big integers, e.g. for cryptography or microsecond-precision timestamps.

`BigInt` type was recently added to the language to represent integers of arbitrary length.  
A `BigInt` value is crented by appending `n` to the end of an integer:
```JavaScript
// the "n" at the end means it's a BigInt
const bigInt = 1234567890123456789012345678901234567890n;
```

As `BigInt` rarely need, it be won't coverd there. See this [BigInt](https://javascript.info/bigint) chapter when you need.

### String
A string in JavaScript must be surround by qoutes.
```JavaScript
let str = "Hello";
let str2 = 'Single qoutes are ok too';
let phrase = `can embed another ${str}`;
```
In JavaScript, there are 3 types of quotes.

1. Double quotes: `"Hello"`
2. Single quotes: `'Hell'`
3. Backtickes: ``Hello``

Backticks are "extended functionality" quotes.  
They allow us to embed variables and expression into a string be wrapping them in `${...}`, for example:
```JavaScript
let name = "John";

// embed a variable
alert( `Hello, ${name}!` ); // Hello, John!

// embed an expression
alert( `the result is ${1 + 2}` ); // the result is 3
```
The expression inside `${…}` is evaluated and the result becomes a part of the string.

> There is no character type.

In some languages, there is a special “character” type for a single character. For example, in the C language and in Java it is called “char”.

In JavaScript, there is no such type. There’s only one type: `string`.
A string may consist of zero characters (be empty), one character or many of them.


### Boolean (logical type)
The boolean type has only two values: `true` and `false`.

```JavaScript
let nameFieldChecked = true; // yes, name field is checked
let ageFieldChecked = false; // no, age field is not checked
```

Boolean values also come as a result of comparisons:
```JavaScript
let isGreater = 4 > 1;
alert( isGreater ); // true (the comparison result is "yes")
```

### This "null" value
The special `null` value does not belong to any of the types described above.

It forms a separate type of its won which contains only the `null` values:
```JavaScript
let age = null;
```
In JavaScript, `null` is not a “reference to a non-existing object” or a “null pointer” like in some other languages.

### The "undefined" value
The special value `undefined` also stands apart.  
It makes a type of its own, just like `null`.  
The meaning of `undefined` is "value is not assigned".  
if a variable is declared, but not assigned, then its value is `undefined`:
```JavaScript
let age;
alert(age); // shows "undefined"
```
Technically, it is possible to explicitly assign `undefined` to a variable:
```JavaScript
let age = 100;
age = undefined;
alert(age); // "undefined"
```
…But we don’t recommend doing that. Normally, one uses `null` to assign an “empty” or “unknown” value to a variable, while `undefined` is reserved as a default initial value for unassigned things.


### Objects and Symbols
The `object` type is special.

All other types are called "primitive" because their values can contain only a single thing.
In contrast, objects are used to store collections of data and more complex entities.

Being that important, objects deserve a special treatment.


### The typeof operator
The `typeof` operator returns the type of the operand. (操作数的类型)
It's useful when we want to process values of different types differently or just want to do a quick check.

A call to `typeof x` returns a string with the type name:
```JavaScript
typeof undefined    // "undefined"
typeof 0            // "number"
typeof 10n          // "bigint"
typeof true         // "boolean"
typeof "foo"        // "string"
typeof Symbol("id") // "symbol"
typeof Math         // "object"   (1)
typeof null         // "object"   (2)
typeof alert        // "function" (3)
```

1. `Math` is a build-in object that provides mathematical operations.
2. The result of `typeof null` is `"object"`. That's an offically recognized error in `typeof`, coming from very early days of JavaScript and kept for compatibility. Definitely, `null` is not an object. It is a special value with a separate type of its own.
3. The result of `typeof alert` is `"function"`, because `alert` is a function.

> The `typeof(x)` syntax

`typeof(x)` is the same as `typeof x`.  
To put it clear: `typeof` is an operator, not a function.
The parentheses here aren't a part of `typeof`.
It's the kind of parentheses used for mathematical grouping, like `(2 + 1)`, but here they contain only one argument `(x)`.

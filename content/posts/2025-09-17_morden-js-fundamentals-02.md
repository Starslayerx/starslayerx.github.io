+++
date = '2025-09-17T8:00:00+08:00'
draft = false
title = 'Morden Javascript Tutorial Chapter 2 - Fundamentals: 06~10'
tags = ['JavaScript']
+++

## 2.6 Interaction: alert, prompt, confirm

Will introduce `alert`, `prompt` and `confirm` in this chapter.

### alert

It shows a message and waits for the user to press "OK".

```JavaScript
alert("Hello");
```

### prompt

This function `prompt` accepts two arguments

```JavaScript
result = prompt(title, [default]);
```

It shows a modal window with a text message, an input field for the visitor, and the buttons OK/Cancel.

`title`: The text to show the visitor.
`default`: An optional second parameter, the initial value for the input field.

> The square brackets in syntax `[...]`

The square brackets around `default` in the syntax above denote that the parameter is optional, not required.

The visitor can type something in the prompt input field and press OK.
Then we get that text in the `result`.
Or they can cancel the input by pressing Cancel or hitting the Esc key, then we get `null` as the `result`.

For instance:

```JavaScript
let age = prompt('How old are you?', 100);
alert(`You are ${age} years old!`); // You are 100 years old!
```

### confirm

The syntax:

```JavaScript
result = confirm(question);
```

The function `confirm` shows a model window with a `question` and two buttons: Ok and Cancel.

The result is `true` if OK is pressed and `false` otherwise.

```JavaScript
let isBoss = confirm("Are you the boss?");
alert( isBoss ); // true if OK is pressed
```

## 2.7 Type Conversions

Most of the time, operators and function automatically convert the values into right type.

For example, `alert` function automatically converts any value to a string to show it.
Mathematical operations convert values to numbers.

### String Conversion

String conversion happens when we need the string form of a value.

We can also call the `String(value)` function to convert a value to a string.

```JavaScript
let value = true;
alert(typeof value); // boolean

value = String(value); // now value is a string "true"
alert(typeof value); // string
```

A `false` becomes `"false"`, `null` becomes `"null"`

### Numberic Conversion

Numberic conversion is mathematical functions and expressions happens automatically.

```JavaScript
alert( "6" / "2" ); // 3, strings are converted to numbers
```

We can use the `Number(value)` function t oexplicitly convert a `value` to a number:

```JavaScript
let str = "123";
alert(typeof str); // string

let num = Number(str); // becomes a number 123
alert(typeof num); // number
```

Explicit conversion is usually required when we read a value from a string-based source like a text form but expect a number to be entered.

If the string is not a valid number, the result of such a conversion is `NaN`.

```JavaScript
let age = Number("an arbitrary string instead of a number")
alert(age); // NaN, conversion failed
```

| Value              | Becomes                                                                                                                                                                                                         |
| :----------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `undefined`        | `NaN`                                                                                                                                                                                                           |
| `null`             | 0                                                                                                                                                                                                               |
| `true` and `false` | 1 and 0                                                                                                                                                                                                         |
| `string`           | Whitespaces (includes spaces, tabs, newline etc.) from the start and end are removed. if the remaining string is empty, the result is 0. Otherwise, the number is "read" from the string. An error gives `NaN`. |

Examples:

```JavaScript
alert( Number("   123   ") ); // 123
alert( Number("123z") );      // NaN
alert( Number(true) );        // 1
alert( Number(false) );        // 0
```

Please note that `null` and `undefined` behave differently here: `null` becomes 0 while `undefined` becomes `NaN`.

Most mathematical operators also perform such conversion.

### Boolean Conversion

Boolean conversion happens in logical operations but can also be performed explicitly with a call to `Boolean(value)`.

The conversion rule:

- Values that are intuitively (直观地) "empty", like `0`, an empty string, `null`, `undefined`, and `NaN`, become `false`.
- Other values become `true`.

```JavaScript
alert( Boolean(1) ); // true
alert( Boolean(0) ); // false

alert( Boolean("hello") ); // true
alert( Boolean("") );      // false
```

> Please note: the string with zero `"0"` is `true`

Some languages (PHP) treat `"0"` as `flase`. But in JavaScript, a non-empty string is always `true`.

```JavaScript
alert( Boolean("0") ); // true
alert( Boolean(" ") ); // spaces, also true (any non-empty string is true)
```

## 2.8 Basic operators, maths

### Terms: "unary", "binary", "operand"

Let's grasp some common terminology:

- An _operand_ is what operators are applied to.  
  For instance, in the multiplication of `5 * 2` there are two operands: the left operand is `5` and the right operand is `2`.  
  Sometimes, people call these "arguments" instead of "operands".

- An _operator_ is unary(单一的) if it has a single operand.  
  For example, the unary negation `-` reverses the sign of a number

  ```JavaScript
  let x = 1;
  x = -x;
  alert( x ); // -1, unary negation was applied
  ```

- An _operator_ is binary if it has two operands.  
  The same minus exists in binary form as well.
  ```JavaScript
  let x = 1, y = 3;
  alert( y - x ); // 2, binary minus subtracts values
  ```

### Maths

The following math operations are supported:

- Addition `+`
- Subtraction `-`
- Multiplication `*`
- Division `/`
- Remainder `%`
- Exponentiation `**`

#### Remainder %

The remainder operator `%`, despite its apperance, is not related to percents.  
The result of `a % b` is the remainder of the integer division of `a` by `b`.

```JavaScript
alert( 5 % 2 ); // 1, the remainder of 5 divided by 2
alert( 8 % 3 ); // 2, the remainder of 8 divided by 3
alert( 8 % 4 ); // 0, the remainder of 8 divided by 4
```

#### Exponentiation \*\*

The exponentiation operator `a ** b` raises `a` to the power of `b`.  
In school maths, we write that as a $a^b$.

```JavaScript
alert( 2 ** 2 ); // 4
alert( 2 ** 3 ); // 8
alert( 2 ** 4 ); // 16
```

The exponentiation operator is defined for non-integer numbers as well.

```JavaScript
alert( 4 ** (1/2) ); // 2
alert( 8 ** (1/3) ); // 2
```

#### String concatenation with binary +

Usually, the plus operator `+` sums numbers.  
But, if the binary `+` is applied to strings, it merges them:

```JavaScript
let s = "my" + "string";
alert(s); // mystring
```

Note that if any operands is a string, then the other one is converted to a string too.

```JavaScript
alert( '1' + 2 ); // "12"
alert( 2 + '1' ); // "21"
```

It doesn't matter whether the first operand is a string or the second one.

Here's a more complex example:

```JavaScript
alert( 2 + 2 + '1' ); // "41" and not "221"
```

Here, operators work one after another. The first `+` sums two numbers, so it returns `4`, then the next `+` adds the string `1` to it, so it's like `4 + '1' = '41'`.

```JavaScript
alert( '1' + 2 + 2 ); // "122" not "14"
```

Here, the first operand is a string, the compiler treats the other two operands as strings too.
The `2` gets concatenated to `1`, so it's like `'1' + 2 = "12"` and `"12" + 2 = "122"`.

The binary `+` is the only operator that supports strings in such a way.
Other arithmetic operators work only with numbers and always convert their operands to numbers.

Here's the demo for subtraction and division:

```JavaScript
alert( 6 - '2' ); // 4, converts '2' to a number
alert( '6' / '2' ); // 3, converts both operands to numbers
```

### Numberic conversion, unary +

The plus `+` exists in to forms: the binary form that we used above and the unary form.

The unary plus or, in other words, the plus operator `+` applied to a single value, doesn't do anything to numbers.
But if the operand is not a number, the unary plus converts it into a number.

For example:

```JavaScript
let x = 1;
alert( +x ); // 1

let y = -2;
alert( +y ); // -2

// Converts non-numbers
alert( +true ); // 1
alert( +"" );   // 0
```

It actually does the same thing as `Number(...)`, but is shorter.

The need to convert strings to numbers arises (发生) very often.
For example, if we are getting values from HTML form fields, they are usually string.
What is we want to sum them?

The binary plus would add them as strings:

```JavaScript
let apples = "2";
let oranges = "3";
alert( apples + oranges ); // "23", the binary plus concatenates (连接, 结合) strings
```

If we want to treat them as numbers, we need to convert and sum them:

```JavaScript
let apples = "2";
let oranges = "3";

// both values converted to numbers before the binary plus
alert( +apples + +oranges); // 5

// longer variant
alert( Number(appels) + Number(oranges) ); // 5
```

### Operator precedence 运算符优先级

There are many operators in JavaScript. Every operator has a corresponding precedence numberl
The one with the lager number executes first.
If we precedence is the same, the execution order is from left to right.

Here’s an extract from the [precedence table](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence)

| **Precedence** | **Name**       | **Sign** |
| :------------- | :------------- | :------- |
| ...            | ...            | ...      |
| 14             | unary plus     | `+`      |
| 14             | unary negation | `-`      |
| 13             | exponentiation | `**`     |
| 12             | multiplication | `*`      |
| 12             | division       | `/`      |
| 11             | addition       | `+`      |
| 11             | subtraction    | `-`      |
| ...            | ...            | ...      |
| 2              | assignment     | `=`      |
| ...            | ...            | ...      |

The "unary plus" has a priority of `14` which is higher than the `11` of "addition".
That's why, in the expression "`+apples + +oranges`", unary pluses work before the addition.

### Assignment

Let's note that an assignment `=` is also an operator.
It is listed in the precedence table with the very low priority of `2`.

That's why, when we assign a variable, like `x = 2 * 2 + 1`,
the calculations are down first and then the `=` is evaluated, storing the result in `x`.

```JavaScript
let x = 2 * 2 + 1;
alert( x ); // 5
```

- **Assignment = returns a value**  
  All operators in JavaScript return a value. That's obvious of `+` and `-`, but also true for `=`.  
  The call `x = value` writes the `value` into `x` and then returns it.

  ```JavaScript
  let a = 1;
  let b = 2;
  let c = 3 - (a = b + 1);

  alert( a ); // 3
  alert( c ); // 0
  ```

  We should understand how it works, because sometimes we see it in JavaScript libraries.  
  Although, please don’t write the code like that. Such tricks definitely don’t make code clearer or readable.

### Chaining assignments

Another interesting feature is the ability to chain assignments:

```JavaScript
let a, b, c
a = b = c = 2 + 2
alert( a ); // 4
alert( b ); // 4
alert( c ); // 4
```

Chained assignments from right to left.
First, the rightmost expression `2 + 2` is evaluated and then assigned to the variables on the left: `c`, `b` and `a`.
At the end, all the variables share a single value.

Once again, for the purposes of readbility it's better to split such code into few lines:

```JavaScript
c = 2 + 2;
b = c;
a = b;
```

That's easier to read, especially when eye-scaning the code fast.

### Modify-in-place

We often need to apply an operator to a variable and store the new result in that some variable.

```JavaScript
let n = 2;
n = n + 5;
n = n * 2;
```

This notation (符号表示; 记号; 标记) can be shortened using the operators `+=` and `*=`:

```JavaScript
let n = 2;
n += 5; // now n = 7 (same as n = n + 5)
n *= 2; // now n = 14 (same as n = n * 2)
alert( n ); // 14
```

Short "modify-and-assign" operators exist for all arithemtical (算术的) and bitwise (位运算的) operations: `/=`, `-=`, etc.

Such operators have the same precedence as a normal assignment, so they run most other calculations:

```JavaScript
let n = 2;
n *= 3 + 5; // right part evaluated first, same as n *= 8
alert( n ); // 16
```

### Increment / decrement

- **Increment** `++` incraeses a variable by 1:

  ```JavaScript
  let counter = 2;
  counter++;
  alert( counter ); // 3
  ```

- **Decrement** `--` decreases a variable by 1:
  ```JavaScript
  let counter = 2;
  counter--;
  alert( counter ); // 1
  ```

> Important:

Increment / decrement can only be applied to variables.
Trying to use it on a value like `5++` will give an error.

### Bitwise operators

Bitwise operators treat arguments as 32-bit integer numbers and work on the level of their binary representation.

These operators are not JavaScript-specific. They are supported in most programming languages.

- AND (`&`)
- OR (`|`)
- XOR (`^`)
- NOT (`~`)
- LEFT SHIFT (`<<`)
- RIGHT SHIFT (`>>`)
- ZERO-FILL RIGHT SHIFT (`>>>`)

### Comma

The comma operator `,` is one of the rarest and most unusual operators.
Sometiems, it's used to write shorter code, so we need to know in order to understand what's going on.

The comma operator allows us to evaluate several expressions, dividing them with a comma `,`.
Each of them is evaluated but only the result of the last one is returned.

```JavaScript
let a = (1 + 2, 3 + 4);
alert( a ); // 7
```

> Comma has a very low precedence

Comma's precedence is lower than `=`, so parentheses `()` are important in the example above.

Without them: `a = 1 + 2, 3 + 4` evaluates `+` first, summing the numbers into `a = 3, 7`, then the assignment operator `=` assigns `a = 3`, and the rest is ignored. It's like `(a = 1 + 2), 3 + 4`

Why do we need an operator that throws away everyting except the last expression?

Sometimes, people use it in more complex constructs to put several actions in one line.

```JavaScript
// three operations in one line
for (a = 1, b = 3, c = a * b; a < 10; a++) {
    ...
}
```

Such tricks are used in many JavaScript frameworks.
That's why we're mentioning them.
But usually they don't improve code readability so we should think well before using them.

## 2.9 Comparisons

Comparsions operators in JavaScript:

- Greater/less than: `a > b`, `a < b`
- Greater/less than or equals: `a >= b`, `a <= b`
- Equals: `a == b`
- Not equals: `a != b`

In this part we'll learn more about different types of comparsions, how JavaScript makes them, including important peculiarities ( 特性) - "JavaScript quirks (小瑕疵, 古怪)"

### Boolean is the result

All compatison operators return a boolean value:

- `true`: means "yes" or "truth"
- `false`: means "no", "wrong" or "not the truth"

```JavaScript
alert( 2 > 1 );  // true (correct)
alert( 2 == 1 ); // false (wrong)
alert( 2 != 1 ); // true (correct)
```

### String comparison

Too see whether a string is greater than another, JavaScript uses the so-called "dictionary" or "lexicographical" (按字母排序的) order.

In other words, strings are compared letter-by-letter.

```JavaScript
alert( 'Z' > 'A' ); // true
alert( 'Glow' > 'Glee' ); //true
alert( 'Bee' > 'Be' ); // true
```

The algorithm to compare two strings is simple:

1. Compare the first character of both strings.
2. If the first character from the first string is greater (or less) than the other string’s, then the first string is greater (or less) than the second.
3. Otherwise, if both strings’ first characters are the same, compare the second characters the same way.
4. Repeat until the end of either string.
5. If both strings end at the same length, then they are equal. Otherwise, the longer string is greater.

> Not a real dictionary, but Unicode order

The comparison algorithm given above is roughly equivalent to the ons used in dictionaries or phone books, but it's exactly the same.

For instance, "a" is greater than "A". Because the lowercase character has a greater index in the internal encoding table JavaScript uses (Unicode).

### Comparison of different types

When comparing values of different types, JavaScript converts the values to numbers.

```JavaScript
alert( '2' > 1 );   // true, string '2' becomes a number 2
alert( '01' == 1 ); // true, string '01' becomes a number 1
```

For boolean values, `true` becomes `1` and `false` becomes `0`.

```JavaScript
alert( true == 1 );  // true
aletr( false == 0 ); // true
```

> A funny consequence

It is possible that at the same time:

- Two values are equal
- One of them is `true` as a boolean and other one is `false` as a boolean.

```JavaScript
let a = 0;
alert( Boolean(a) ); // false

let b = "0";
alert( Boolean(b) ); // true

alert( a == b ); // true!
```

### Strict equality

A regular equal check `==` has a problem.
It cannot different `0` from `false`.

```JavaScript
alert( 0 == false ); // true
```

The same thing happens with an empty string:

```JavaScript
alert( '' == false ); // true
```

This happens because operands of different types are converted to numbers by the equality operator `==`.
An empty string, just like `false`, becomes a zero.

**A strict equality opertaor `===` checks the equality without type conversion.**

```JavaScript
alert( 0 === false ); // false, because the types are different
```

There is also a "strict non-equality" operator `!==` analogous 类似 to `!=`.

The strict equality operator is a bit longer to write, but makes it obvious what's going on and leaves less room for errors.

### Comparison with null and undefined

There's a non-intuitive behavior when `null` or `undefined` are compared to other values.  
**For a strict equality check `===`**

```JavaScript
alert( null === undefined );  // false
```

**For a non-strict check `==`**  
There's a special rule. These two are a "sweet couple": they equal each other (`==`), but not any other value.

```JavaScript
alert( null == undefined ); // true
```

**For maths and other comparisons `<` `>` `<=` `>=`**  
`null/undefined` are converted to numbers: `null` becomes `0`, while `undefined` becomes `NaN`.

Here are some funny things.

### Strage result: null vs 0

Let's compare `null` with a zero:

```JavaScript
alert( null > 0 );  // (1) false
alert( null == 0 ); // (2) false
alert( null >= 0 ); // (3) true
```

The reason is that an equality check `==` and comparsions `> < >= <=` work differently.
Comparisons convert `null` to a number, treating it as `0`.
That's why (3) `null >= 0` is true and (1) `null > 0` is false.

On the other hand, the equality check `==` for `undefined` and `null` is defined such that,
without any conversions, they equal each other and don't equal anyting else.
That's why (2) `null == 0` is false.

```JavaScript
alter( null == undefined );  // true
alert( Boolean(undefined) ); // false, null 在宽松比较中只等于 undefined 和 自身
alert( Boolean(null) );      // false
```

- null 和 undefined 在宽松相等比较中, 只彼此相等, 不与任何其他除了自身值相等.

### An incomparable undefined

The value `undefined` shouldn't be compared to other values:

```
alert( undefined > 0 ); // false (1)
alert( undefined < 0 ); // false (2)
alert( undefined == 0 ); // false (3)
```

We get these results because:

- Comparsion `(1)` and `(2)` return `false` because `undefined` gets converted to `NaN` and `NaN` is a special numeric value which returns `false` for all comparisons.
- The equality check `(3)` returns `false` because `undefined` only equals `null`, `undefined`, and no other value.

**Avoid problems**

- Treat any comparison with `undefined/null` except the strict equality `===` with exceptional care.
- Don't use comparisons `>= > < <=` with a variable which may be `null/undefined`, unless you're really sure of what you're doing. If a variable can have these values, check for them separately.

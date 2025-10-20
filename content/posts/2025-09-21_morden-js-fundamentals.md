+++
date = '2025-09-21T8:00:00+08:00'
draft = true
title = 'Morden Javascript Tutorial Part 3- Fundamentals: 11~15'
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
    alter('truthly!');
}
```

#### OR "||" finds the first truthly value
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

+++
date = '2025-09-16T8:00:00+08:00'
draft = true
title = 'Morden Javascript Tutorial - Fundamentals :06~10'
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







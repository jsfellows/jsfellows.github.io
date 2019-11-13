---
layout: post
title:  "Arrow functions vs. Common functions"
author: viniciusls
categories: [ javascript, es6, nodejs, vanillajs ]
image: assets/images/2.png
featured: true
hidden: false
---

Do you know the difference between *Arrow functions* and *Common functions* in JavaScript? Let's talk about that!

Hi JSFellows, how are you? I hope you're fine! Let's go for the first post of the blog. I was hoping to do it earlier right after I finished installing all needed things (template and other internal details) but I had to focus on studying lots of things in order to start bringing great contents and ideas to this space.

So the first subject that I choose to start is something that it's not so new but I feel that all new (or even experienced, which probably went out for 5 minutes for a coffee break and when came back to work saw that JS changed everything just like that) developers: Arrow functions.

My promise here is to be as simple as possible in subjects like this one, specially to avoid confusing you if you're not comfortable with this kind of function yet. So basically arrow function is a different syntax of declaring a function. It can do exactly what a common function does but in a (more) summarized way (or less verbose). It also has a crucial difference in comparison with a common function: the this context.

#### Syntax

Let's compare some differences in the syntax between arrow functions and common functions.

Syntax of a regular function:

```javascript
function <function_name>(<param1>, <param2>, ..., <paramn>) {
    // body of the function
}
```

Using this syntax, a real function would be something like this:

```javascript
function sum(num1, num2) {
     return num1 + num2;
}
```

Let's have a look on the arrow function syntax:

```javascript
<function_name>(<param1>, <param2>, ..., <paramn>) => {
    // body of the function
}
```

Using this syntax for the example function would result in something like this:

```javascript
sum(num1, num2) => {
    return num1 + num2;
}
```

However, there's a small trick that arrow functions allow us to do. If your code block has only one line, which would return something, you can use it without needing { }. So this last example would become even smaller:

```javascript
sum(num1, num2) =>  num1 + num2;
````

But remember: this last syntax will only work if you just need to perform an one-line action and return the result of it. It's kinda of an implicit return.

These functions that I wrote can either be assigned to a variable (and the function name will be abstracted to the variable name) like:

```javascript
const sum = (num1, num2) => num1 + num2;

sum(1,2); // would return 3
```

Or you can use it as callback functions like this:

```javascript
request.on('data', (data) => {
    console.log(data);
});
```

Or it can be used inside a class in the exactly same form I wrote, like this:

```javascript
class Mathematic {
    sum(num1, num2) => num1 + num2;

    increaseByPercent(value, percent) => {
        const increase = value * (percent/100);

        return value + increase;
    }
}
```

Talking about *classes*...

#### Use of *this*

If you're a beginner, you're probably not familiar with the this keyword and its use. Or maybe you have already saw it on other language like Java and PHP, where this is a lot more common due to the Object-Oriented programming structure (or at least due to the class programming model). In a class model, used mostly in Object-Oriented programming, this is used to reference context. So if this is used inside a class method, it'll refer the class instance; on the other hand if this is used on a closure or anonymous function, it'll refer the closure or function itself. In other words, in common functions this will refer the object that called the function, while in arrow functions this will refer the object that defined the arrow functions (in my example it was a class so this will refer the class). Talking about classes, we call this to use class variables or other class methods.

# Under construction

This post is under development. Come back soon to read the rest of it.


#### Conclusion
So, that's it! I hope I could help you to understand (more) about this cool functionality that comes to our beloved JavaScript. Feel free to ask me anything about this subject or any other subject that you wanna see here in this space using the comments section below :) See ya!
---
layout: post
title: 'Hint: Global Initialization Order'
date: 2022-06-18 19:30:00 GMT+3
categories: [Notes, Hints]
tags: [Notes, C++]
---

Global initialization order is rather trivial thing, but anyone encounters mistakes related to it one day or another.
I've decided to post this hint, that I would've given to my past self. Lets start from the problem definition.

### Problem

Lets imagine that we have these two static fields:

```c++
class MyHolderClass final
{
    static AClass aObject;

    static BClass bObject;
};
```

It doesn't change things whether they are inside one holder class, two different ones or are top level citizens.

There is one possible problem: `aObject` initializer might access `bObject`, but `bObject` is not yet initialized, 
or vice versa. Therefore, access to uninitialized memory occurs and what happens then no one can predict:
behaviour varies from blunt crashes to subtle bugs.

I guess, everyone sooner or later encounters bugs related to this problem. And everyone who encountered them knows
that it is pain in the ass to debug them. So, how to avoid this problem?

### Solution

Thankfully, there is such thing as function-scope static variables, that are initialized during first call to the 
function. Of course, they have small impact on performance (close to one additional `if` per call, I guess),
but they allow us to forget about initialization order problems once and for all.

```c++
// Header file.

class MyHolderClass final
{
    static AClass &GetA ();

    static BClass &GetB ();
};

// Object file.

AClass &MyHolderClass::GetA ()
{
    static AClass aObject;
    return aObject;
}

BClass &MyHolderClass::GetB ()
{
    static BClass bObject;
    return bObject;
}
```

In this case, function call always guarantees that `aObject` and `bObject` are initialized by the time they are
accessed. And if their initializators depend one on another, we will receive easily debuggable stackoverflow crash.

### When this solution is bad

- If you're perfectly sure that static global variables are never accessed during global initialization, it's better
  to avoid packing them inside functions, because by packing you might reduce readability and introduce additional
  performance cost to variable access.

- If your hot path code accesses this global variable and needs to achieve top notch performance, it is unwise
  to add one unnecessary `if` to the equation (and possibly add one function call too, if LTOs are not smart enough).

So, choose wisely! :)

This post might sound really trivial, but you shouldn't just ignore this theme. I've encountered mistakes like 
this in some codebases (hello Vizor Games mobile department!) and it wasn't very pleasant.

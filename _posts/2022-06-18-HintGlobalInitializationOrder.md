---
layout: post
title: 'Hint: Global Initialization Order'
date: 2022-06-18 19:30:00 GMT+3
categories: [Notes, Hints]
tags: [Notes, C++]
---

Global initialization order is a rather trivial thing, but anyone encounters mistakes related to it some day or another.
I've decided to post the hint that I would've given to my past self. Let's start from the problem definition.

### Problem

Let's imagine that we have these two static fields:

```c++
class MyHolderClass final
{
    static AClass aObject;

    static BClass bObject;
};
```

It doesn't matter whether they are inside one holder class, two different ones or are top level citizens.

There is one possible problem: `aObject` initializer might access `bObject`, but `bObject` is not yet initialized,
or vice versa. Therefore, access to uninitialized memory occurs and no one can predict what happens then:
behaviour varies from blunt crashes to subtle bugs.

I guess, everyone sooner or later encounters bugs related to this problem. And everyone who did knows
that it is a pain in the ass to debug them. So, how to avoid this problem?

### Solution

Thankfully, there is such thing as function-scope static variables, that are initialized during the first call to the
function. Of course, they have a small impact on performance (close to one additional `if` per call, I guess),
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

In this case, the function call always guarantees that `aObject` and `bObject` are initialized by the time they are
accessed. And if their initializers depend on one another, we will receive easily debuggable stackoverflow crash.

### When this solution is bad

- If you're completely sure that static global variables are never accessed during global initialization, it's better
  to avoid packing them inside functions, because this way you might reduce readability and introduce additional
  performance cost to variable access.

- If your hot path code accesses this global variable and needs to achieve top-notch performance, it is unwise
  to add one unnecessary `if` to the equation (and possibly add one function call too, if LTOs are not smart enough).

So choose wisely! :)

This post might sound really trivial, but you shouldn't just ignore this topic. I've encountered mistakes like
this in some codebases (hello Vizor Games mobile department!) and it wasn't very pleasant.

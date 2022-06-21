---
layout: post
title: 'Emergence: What & Why?'
date: 2022-06-18 17:00:00 GMT+3
categories: [Emergence, Development Log]
tags: [Emergence, C++, GameDev]
pin: true
---

I've been developing [Emergence](https://github.com/KonstantinTomashevich/Emergence) project for more than a year.
Or two years, if you count a previous unfortunate attempt!
I've decided to finally come out of the shadow and start writing about it.
Today I am going to talk on the decision to start this project.

This is my first post on this blog, therefore I think it's a good idea to share what kinds of posts I plan to do later:
- Emergence Development Log: Everything about [Emergence](https://github.com/KonstantinTomashevich/Emergence) project
  development.
- Emergence tutorials: Tutorials on how to work with [Emergence](https://github.com/KonstantinTomashevich/Emergence)
  libraries once they're ready, of course. :)
- Hints: Small notes about different aspects of programming.
- Arrow to the Knee: name speaks for itself, I guess. These notes will be about shooting yourself with your code.
- There might be articles and tutorials on general topics, connected to game development, but I'm not sure yet.

If you'd like to get notifications about new posts, subscribe to my
[Telegram channel](https://t.me/WhatKindOfDevAreYou). :)

### Motivation

I discovered data-driven programming when I joined [BattlePrime](https://www.battleprime.com/) team.
These guys were really cool professionals, and I'm glad I was working with them.
After years of object-oriented development I was really excited to try something new and quickly became a true ECS zealot.
But there were a lot of flaws in our ECS design too, not because people were unprofessional, but because designing
architecture is generally difficult.
From my perspective, so called flag-components were one of the main problems in our design.
We had `AliveComponent`, `DeadComponent`, `ActiveAvatarComponent`, `ControlledByPlayerComponent` and so on.
These components had no data and were only used to filter entities through components masks.
I was thinking about this a lot and one question was arising on and on: why can't we just use query-like API instead of
classic ECS entity families?

I was digging further and further into this `why not queries?` question and started believing in database-driven
ECS concept.
Yes, it's not new and many people before me tried to implement it and failed.
But this idea was still very interesting and motivating for me.
I had a lot of questions.
What is the difference between receiving a component from entity and quering by entity id?
Why do we even need entities, when everything can be records, loosely connected by indexed queries?
So, I decided to write my own ECS framework with database-like style and ~~prostitutes~~ queries.

Also, I was unsatisfied with some concepts that were highly used in [BattlePrime](https://www.battleprime.com/) and
some other ECS frameworks. Therefore, I decided to stick to some additional rules while writing
[Emergence](https://github.com/KonstantinTomashevich/Emergence):

- There should be no `virtual` methods unless they are required by third party libraries. Not only are `virtual`s
  kind of slow from a performance point of view, but also a source of unknown behaviour. By allowing to put almost
  everything into everything, `virtual`s might lead to strange bugs and unnecessary complications. I decided that I
  want everything to be straightforward, so there should be no `virtual`s.

- Template usage should be minimized. There are some people out there who really really love templates, but usually
  they ignore one significant drawback: compile time. One engineer in [BattlePrime](https://www.battleprime.com/) team
  calculated that including `<core/entity.h>` increases unit compile time by roughly 6 seconds. And `<core/entity.h>`
  was, obviously, included everywhere. In my opinion, long compile times make programming unbearable, therefore I
  decided to limit template usage in order to have a reasonable compile time.

- Multithreaded code must be planned to be multithreaded from the beginning, not ad-hooked to be multithreaded after.
  One of the main advantages of data-driven design is that it is usually much easier to adapt it for parallel execution
  comparing to a classic OOP approach. Unfortunately, [BattlePrime](https://www.battleprime.com/) team was never able
  to make their ECS truly parallel: they tried several times, but failed. One of the biggest problems was that their
  ECS had never been designed to be multithreaded in the first place, so I decided to think about multithreading
  from the start.

There was also one additional thing I wanted to try out during
[Emergence](https://github.com/KonstantinTomashevich/Emergence) development: link time polymorphism. It sounded like
something unbelievable: polymorphism without runtime performance and compile time drawbacks.

### Goals

After failing my previous attempt -- Temple -- I've decided that I need to understand better what I am writing before
starting again from scratch.
I was planning to create several disconnected or loosely connected modules and build my own ECS on top of them.
And that's exactly why I called this project Emergence: my ECS framework would emerge on top of different disconnected
libraries, like consciousness on top of our neurons in emergent consciousness philosophy. Below I am going to provide
the list of the libraries I've planned:

- CMake framework for link time polymorphism. I've planned many libraries and needed good support from the build
  system for my beloved link time polymorphism. :)

- Memory allocation library with pools, stacks, unique strings and hierarchical memory profiling. While the first ones
  are quite easy to implement, hierarchical (aka "build memory usage flame graph for me, bitch") memory profiling
  has been quite tricky, because I haven't been able to find any opensource tools that can do that.

- Simplistic field-only reflection that also knows about constructors and destructors. I needed reflection for
  component indexing, and I didn't want to use any of the existing heavyweight libraries, therefore I decided to create
  my own lightweight library.

- Logging library, possibly a wrapper for some logging framework. How can you track bugs without logging?

- Utility for rendering complex runtime structures into DOT graphs. It is always useful to render your system graph or
  some other data.

- Indexed object storage. Just a simple collection of objects with one no-so-simple feature: it must provide user
  with an opportunity to create hash indices, sorted indices and volumetric (aka bounding box) indices.

- Library for managing indexed object storages that also supports unindexed storages and singletons. Sounds exactly
  like ECS World, isn't it?

- Library for registering and executing parallel tasks that use shared resources and depend on each other. Also,
  this library needs to detect circular dependencies and race conditions. Later it is going to be executing systems inside
  my ECS framework.

- And finally, the pinnacle of the human development, my ECS framework. :D

### What is it now?

Almost a year and three months have passed since I started developing
[Emergence](https://github.com/KonstantinTomashevich/Emergence), and I'm happy to say that almost all the goals above
have been achieved. There is just one thing left to be done: I need to develop a small proof-of-concept project to check
whether my ECS framework is good at doing what it needs to do. But, as it always happens, this last thing is
much more complex than it sounds: to finish this project, I need to at least integrate physics and graphics. And
I've already finished small PhysX and Urho3D integrations, but who knows what other integrations I might need later.

I hope to finish proof-of-concept in the next month or two and finally be able to present my beloved ECS framework
to the public. Let's just hope that there won't be any complex obstacles in the process of doing so.

That's all for now. Hope you've enjoyed reading this article. :)

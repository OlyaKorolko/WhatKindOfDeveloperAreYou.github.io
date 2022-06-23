---
layout: post
title: 'Emergence: Why is there no entities in Celerity?'
date: 2022-06-23 12:40:00 GMT+3
categories: [Emergence, Development Log]
tags: [Emergence, ECS, GameDev]
---

Entity is the first of three core principles of ECS design pattern. Nevertheless, I've decided that there should be
no entities in [Emergence](https://github.com/KonstantinTomashevich/Emergence) ECS-like framework Celerity. 
I will tell you how I've arrived at this decision in this post.

### ECS pattern

For those who are not familiar with Entity-Component-System pattern I decided to write a quick recap.

Historically, ECS is an evolution of [Components](https://gameprogrammingpatterns.com/component.html) pattern,
in which parts of game world, usually called game objects or actors, can receive additional features through
addition of components with custom logic to them. This pattern saved quite a few projects from drawbacks of malicious
"just write game logic and use inheritance" pattern, used extensively in old times. That's the reason why
[Components](https://gameprogrammingpatterns.com/component.html) is a core feature of both well-known engines, like
Unity and Unreal Engine, and less known ones, like Urho3D.

But [Components](https://gameprogrammingpatterns.com/component.html) pattern didn't solve all pitfails of its 
predecessor, and one of the most malicious problems was update order. When every object and every component
chaotically subscribes to update routine, it is not only difficult to know whether it uses current frame or previous
frame state of other objects, but also nearly impossible to track what happens during game loop. This kind of chaos
is a problem by itself, but it leads to another problem too: how to make gameplay logic parallel when everything is
so chaotic and uncontrollable?

That's where Entity-Component-System comes to the rescue by removing all logic from components and returning back
to the times of procedural programming. Sounds too dramatic and old-fashioned? Maybe, but it really helped to
sort things out, so lets describe what it is. Lets start from describing 3 core pillars of ECS:

- **Entity**: it is something, that exists inside game world. It might be a tree, might be a player, might be an 
  invisible controller. But entity by itself has no logic at all, it is just a collection of components! Nothing more,
  nothing less.

- **Component**: it is just a collection of data, possibly even a plain-old-data structure. Ideally, components have
  no logic at all, just data layout. But sometimes there are utility functions that help to access data or modify it
  without breaking coherency.

- **System**: it is a place to put game logic into. It might be a function or something more sophisticated. The core
  rule is that all meaningful logic should be placed inside systems and that systems depend on each other to make
  update order predictable.

So, what problems does this pattern solves?

- Update order is always predictable due to dependencies between systems, that contain all the logic.
- It is much easier to make logic parallel, because systems access well-known sets of components, so we can
  just execute systems simultaneously if they do not both modify one component type.
- It is much easier to serialize data and send it over the network, because logic is separated from data.

Now you know what ECS theoretically is, so we can discuss the topic of entities deeper.

### What is Entity?

Many old-school developers still think of entities as game objects without logic: just game objects that store
multiple components. It is not incorrect to think like that, but, in my opinion, it hides storage-management
aspect of ECS by simply treating entities as vectors.

Modern approaches usually treat entities as just numbers. Yes, simple numbers that are used to query components 
from their component storages. For example, we can have separate memory pool for each component type, so access
to several components of the same type will be much more cache coherent. Of course it is not the only "correct"
way to treat entity concept: there are things like entity family locality optimizations, but I won't dig further
into that topic in this post as I'm not an expert in that. 

But lets dig deeper into "entity as number" idea. Doesn't it remind you about some other topics from computer 
science? Like foreign indices from relational databases? The thing is that we can view component types as tables
and entities as foreign keys, that create relations between components. I like finding parallels between different
computer science topics and therefore I really like this parallel: it looks like a powerful path to improve
ECS pattern even further by adding ability to create other indices instead of only quering by entity id.
It is not a fresh idea, but I've never seen successful implementations of that principle, therefore I decided
that I must try to implement it myself.

### How additional indexing helps?

At first glance additional indices might sound excessive. Why do we need them after all if everything worked fine 
without them? But I'm convinced that there are enough cases to justify that we need them:

- In some projects there're a lot of so called flag components, like `AliveComponent`, `ReadyToSpawnComponent`, etc.
  These components have no data and are only used to filter entities. But if we were able to create additional indices,
  we wouldn't need these components any more: we could just store this data as fields, like `bool alive`, and create
  indices for this fields. Need to iterate over all alive units? Just use query on this index.

- I've seen a lot of constructions like `if (timeNow < component->coolingDownUntil) continue;`. It's not only bad for
  performance, but also decreases readability by adding checks like this everywhere. But why don't create sorted index
  on `coolingDownUntil`? It would allow us to get cooled down entities fast and without iterate-over-everything
  overhead. And because cooldown is not really applied every frame, index usage overhead will be really tiny.

- Almost always there are some so-called managers, that accompany game world. For example, `ConfigManager` or 
  `WeaponStatsManager`. Usually, these managers work as glorified hash maps and this made me think: why don't put
  their data as additional tables inside game world? We can use same API to create indices over configs and stats
  ids and get rid of excessive managers.

### Game world as universal storage

By allowing users to create additional tables and indices for whatever they need, we convert game world into some
kind of universal storage. We can store a lot of things there:

- Components, first and foremost.
- Records with short lifetime, like events. Just don't index them. :)
- Singletons.
- Configs.
- Stats.
- Maybe even assets!

It might sound messy, but it also sounds really powerful, isn't it? At least it is for me, therefore I decided to
make it the core idea for my ECS implementation. Sticking to this idea makes principle of entities obsolete: 
we just have tables with records and systems (personally, I prefer to call them tasks) that operate on this records.

Of course, this approach has its pros and cons. Lets start from pros:

- User can store everything in game world and treat all data in the same way.
- Indices can be created for whatever user needs without restrictions.

And there are cons:

- Entity family optimizations can not be applied here, because there are no entities.
- It might be difficult to orient in storage, because it will store almost everything.

For me, pros outweigh the cons, but you're free to argue about it.

In the end, I'd like to share some example from my unfinished demo to show how this works.

#### Inserting physical material into world

```c++
// Tasks (systems) store prepared queries as fields. This query allows user to insert records.
Emergence::Celerity::InsertLongTermQuery insertDynamicsMaterial;
// ...

/// Tasksk initialize queries in constructors.
: insertDynamicsMaterial (INSERT_LONG_TERM (Emergence::Physics::DynamicsMaterial)),
// ...

// Then we can insert materials using this query during task execution.
auto materialCursor = insertDynamicsMaterial.Execute ();
auto *material = static_cast<Emergence::Physics::DynamicsMaterial *> (++materialCursor);

material->id = PhysicsConstant::DEFAULT_MATERIAL_ID;
material->staticFriction = 0.4f;
material->dynamicFriction = 0.4f;
material->enableFriction = true;
material->restitution = 0.3f;
material->density = 400.0f;
```

### Quering by entity (object) id

```c++
// This query allows user to gain read-only access to all records with 
// requested value in given field.
Emergence::Celerity::FetchValueQuery fetchShapeByObjectId;
// ...

: fetchShapeByObjectId (FETCH_VALUE_1F (CollisionShapeComponent, objectId)),
// ...

// Process all shapes on object using prepared query and cycle.
for (auto shapeCursor = fetchShapeByObjectId.Execute (&objectId);
     const auto *shape = static_cast<const CollisionShapeComponent *> (*shapeCursor); ++shapeCursor)
{
    // ...
}
```

### Querying cooled down components

```c++
// This query allows user to gain full write access to all records 
// where field value is inside given interval.
Emergence::Celerity::ModifyAscendingRangeQuery modifyShootersByCoolingDownUntil;
// ...

: modifyShootersByCoolingDownUntil (MODIFY_ASCENDING_RANGE (ShooterComponent, coolingDownUntilNs)),
/// ...

// Process all shooters where colldown ended before this moment.
// `nullptr` as query argument represents negative (if first) or positive (if second) infinity.
for (auto shooterCursor = modifyShootersByCoolingDownUntil.Execute (nullptr, &time->fixedTimeNs);
     auto *shooter = static_cast<ShooterComponent *> (*shooterCursor);)
{
    // ...
}
```

That's all for now. Hope you've enjoyed reading! :)

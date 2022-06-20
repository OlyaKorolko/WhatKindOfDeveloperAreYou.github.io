---
layout: post
title: 'Arrow to the Knee: Editing unordered_multiset items'
date: 2022-06-20 12:20:00 GMT+3
categories: [Notes, Arrow to the Knee]
tags: [Arrow to the Knee, C++, Containers]
---

Have you ever thought of editing `std::unordered_multiset` items while iterating over it? Does it even make sense?
In this post I will tell you why and how I tried to do it and what happened to my code afterwards.

### Getting into position

`std::unordered_multiset` iterator returns const references to items and it works like that for a reason: developers
never expected that some phycho would try to edit items during iteration. And even if they did expect it,
the is no practical way to make this work, because there is no cheap way to teach `std::unordered_multiset`
to track changes in its records.

So, why did I need to edit items, after all? This case arised during 
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus)
development: user requests write access to record and we need to provide this access as fast as possible and only after
user has done everything needed to be done we are checking whether key fields are different. Of course, to do this check
we also have separate preallocated record that stores only key fields. So, when we know that user has changed key 
fields, we need to somehow update `std::unordered_multiset` according to these changes.

There is one more case which is even more complicated from the first glance: user might just delete record and in this
case we don't want to check whether key fields were changed, we want to just erase this record right away.

### Getting shot

The first thought that came to my mind was: 

> Oh, we already have iterators, so we can just erase and then emplace and everything will be fine!
{: .prompt-danger }

Sounds reasonable, right? Even if value is not the same as it was before, we still have valid iterator,
so any operations with it should work fine. But it isn't the case with unordered containers.

The thing is, just having the iterator is not enough to perform the erasure properly: during erasure unordered
container needs to calculate hash once more and that is where things are getting fucked up, because hash is 
different now.

To check, try running this snippet using any compiler and any STD implementation you like:

```c++
std::unordered_multiset <int> set;
set.emplace (1);
set.emplace (2);
set.emplace (3);
    
auto iterator = set.find (2);
const_cast <int&> (*iterator) = 4;
set.erase (iterator);
    
for (const int &number : set)
{
    std::cout << number << std::endl;
}
```

You can also try to execute it with numbers other than `4`: it would result in unexpected 
~~(or, sometimes, normal, but not really expacted)~~ behaviours.

### Healing the wounds

But in 
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus)
everything works fine, so there is solution for this problem, right? Yep, there is! A bit clunky and ad hok though.

We can start from the fact that we need to somehow keep hash the same after changing key fields. That means, that
we need to roll key fields back to their initial values, right? This is one way to think about that. And it is close
to how
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus)
actually deals with this problem: it stores multiset of pointers and it already has preallocated record with initial
values for change checking, therefore we can just edit iterator like that:

```c++
// Copy link to the record for further usage.
const void *record = *_position;

// Replace iterator content with backup to make hash result valid.
const_cast<void const *&> (*_position) = storage->GetEditedRecordBackup ();

// Now we can safely erase record from our set.
records.erase (_position);

// And reinsert our changed record!
records.emplace_back (record);
```

Code above is a simplified version of what
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus)
does.

But what if you don't have record backup? Well, if you know index count during compile time, which is not the case for
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus)
, there is another way do deal with this problem: storing hash as additional field and recalculating it only after
record is erased from the map. Unfortunately, I've never tried it myself, so I can not tell you more about this method.

Hope you've enjoyed reading about this tragic failings with `std::unordered_set`! :)

---
title: "Notes on formal languages: alphabets, strings and languages"
date: 2023-08-08T00:00:00-03:00
categories: ["formal-languages", "today-i-learned"]
draft: false
---

An `alphabet` is any set of finite symbols such as `a` and `b`. For example, the alphabet `Σ = {a, b}` is an alphabet that contains the `strings` that can be built by combining `a` and `b` and the alphabet `Σ = {0, 1}` is the an alphabet that contains the strings that can be built by combining `0` and `1`.  

Symbols such as `a` and `b` put together to form something like `bbaa` are called `strings`.
> `bbaa` is a `string` built from the `symbols` `a` and `b`

Alphabets and symbols can be used to create a `language` which is a set of strings over some fixed alphabet.  

Given the language `L = x ∈ {0, 1}* | x starts_with(x, 10)`:

```
S -> 1A
A -> 10B
B -> 10B | 1B | 0B | ε
```

We can try to match a few strings:

```
10 (Matches)
S -> 1A
A -> 10B
B -> ε

01 (Doesn't match)
S -> No match

101 (Matches)
S -> 1A
A -> 10B
B -> 1B
B -> ε
```

> `ε` is the empty string and it is being used to stop the recursion.

Some of this comes from one of the books I'm reading at the moment: [FORMAL LANGUAGE: A PRACTICAL INTRODUCTION](https://fbeedle.com/our-books/15-formal-language-a-practical-introduction-9781590281970.html)
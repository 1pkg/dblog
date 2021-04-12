---
author: 'Kostiantyn Masliuk'
title: 'Implementing go ambiguous operator'
date: '2021-04-11'
---

Lately, I discovered one great tech blog https://tpaschalis.github.io. One of the articles that caught my eye was [amb operator in golang](https://tpaschalis.github.io/golang-amb-operator/). In my opinion, ambigious operator is brilliant concept in scope of elegant functional programming. Ambiguous operator is defined by its author John McCarthy as

```
Ambiguous functions: Functions whose value are incompletely specified.
May be useful in providing facts about functions where certain details are irrelevant to the statement being proved.
```

Although the real world is harsh to to this operator, it rarely can be used due to the computation complexity. Generally `amb` operator requires matching all possible combination of sets of provided ambiguous variables. Which leaves us with polinomial complexity `n^m`, where `n` - size of ambiguous variable and `m` - number of ambiguous variables.

Back to the article, where author implemented really specific version of `amb` operator with help of backtracking that matches ambiguous strings only. Looking at it, I decided to reimplement `amb` operator in go by myself to make it more generic and remove unnecessary concurency manipulations. In result of this, I created generic go `amb` operator implementation and wrapped it into the library [gamb](https://github.com/1pkg/gamb). `gamb` operates on `interface{}` isntead of any concrete type and exposes ambiguous expression predicate directly.
While playing with `gamb`, I've also discovered different "flavors" that can be applied to `amb` operator:

- `amb` operator that finds first matching variable (`Amb` in [gamb](https://github.com/1pkg/gamb/blob/master/amb.go#L17));
- `amb` operator that finds all matching variables (`All` in [gamb](https://github.com/1pkg/gamb/blob/master/amb.go#L49));
- `amb` operator that finds first matching variable in strict single pass order (`Ord` in [gamb](https://github.com/1pkg/gamb/blob/master/amb.go#L84));

All and all, an implementation of `amb` operator is pretty straightforward, you can chose between backtracking or looping (I tried both). The only intresting part located inside looping implementation, the trick - hot to roll out n-unbound loops into recursion.

In conclusion, `amb` operator was fine discovery and exercise to me. Though `amb` is probably, not applicable to most of real life problems. Still, the more you know, the better.

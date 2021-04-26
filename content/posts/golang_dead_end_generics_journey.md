---
author: 'Kostiantyn Masliuk'
title: "Golang dead end generics journey"
date: '2021-04-25'
---

As [the proposal](https://github.com/golang/go/issues/43651) of adding generics in go has finally came to its endspiel and has been accepted. I want to add my two cents on this event and explain why I think that generics is a bad idea how to introduce a meta programming to go.

## Origins

The history of generics in go is truly amazing. The ultimate answer to the question of having generics in go has evolved together with the language.

Originally, go official [FAQ](https://golang.org/doc/faq#generics) answered it in the following way.
```
Why does Go not have generic types? Generics may well be added at some point.
We don't feel an urgency for them, although we understand some programmers do.
Generics are convenient but they come at a cost in complexity in the type system and run-time.
We haven't yet found a design that gives value proportionate to the complexity,
although we continue to think about it. Meanwhile, Go's built-in maps and slices,
plus the ability to use the empty interface to construct containers (with explicit unboxing)
mean in many cases it is possible to write code that does what generics would enable,
if less smoothly. This remains an open issue.
```

But the answer has recently been updated and now [FAQ](https://golang.org/doc/faq#generics) contains.
```
A language proposal implementing a form of generic types has been accepted
for inclusion in the language. If all goes well it will be available in the Go 1.18 release.
Go was intended as a language for writing server programs that would be easy to maintain over time.
(See this article for more background.) The design concentrated on things like scalability,
readability, and concurrency. Polymorphic programming did not seem essential to the language's goals
at the time, and so was left out for simplicity. The language is more mature now, and there is scope
to consider some form of generic programming. However, there remain some caveats.
Generics are convenient but they come at a cost in complexity in the type system and run-time.
We haven't yet found a design that gives value proportionate to the complexity, although we continue
to think about it. Meanwhile, Go's built-in maps and slices, plus the ability
to use the empty interface to construct containers (with explicit unboxing) mean in many cases
it is possible to write code that does what generics would enable, if less smoothly.
The topic remains open. For a look at several previous unsuccessful attempts to design a good
generics solution for Go, see this proposal.
```

## Analyzing the community

Let's take a deeper look on how we got there. It all started since day 0 of go public appearance [in 2010](https://en.wikipedia.org/wiki/Go_(programming_language)) with, in my opinion, the smart choice to not include generics in the initial language release. Soon enough, go conquered some popularity and developed community around. Along with this came endless complaints about lack of generics in go and related comparisons with Java and other languages alike. I believe, this is nicely explained by constant migration of new developers from other languages and their desire to make go look like mainstream language `X`.

To be 100% sure, though, let's also analyze what people wanted and what they asked for exactly. Go is awesome; it has yearly official survey ritual to gather community opinion on various vital questions. Sad part though, there was not public go survey until year of 2016. Though, probably, it's not that hard to find hundreds of articles with outcry complaining about lack of generics in go, daiting all the way back to the go release date. Still, we will use just public survey results as the source of the analyze. Once again, the data is a bit "fresh" and doesn't fully represent the full historical perspective of generics bargaining, however, it verifies the fact that generics desire didn't vanish within passing years.

[2016](https://blog.golang.org/survey2016-results) generics are the first on `What changes would improve Go most?`
![survey 2016 results](/survey_2016.png)

[2017](https://blog.golang.org/survey2017-results) generics are the second?! (no, the first) on `What changes would improve Go most?`
![survey 2017 results](/survey_2017.png)

[2018](https://blog.golang.org/survey2018-results) surprisingly, generics are the third on `What is the biggest challenge you personally face using Go today?` (generics are only the third, probably, due to the question rephrasing)
![survey 2018 results](/survey_2018.png)

[2019](https://blog.golang.org/survey2019-results) generics are the first (beating the second "better error handling" ~ 4/1) on `Which critical features do you need which are not available in Go?`
![survey 2019 results](/survey_2019.png)

[2020](https://blog.golang.org/survey2020-results) generics are the first (this time difference with "better error handling" not that drastic) on `Which critical features do you need which are not available in Go?`
![survey 2020 results](/survey_2020.png)

One extra insightful source that is worth some attention - various official proposals around generics programming in go.
 - [Go should have generics](https://github.com/golang/proposal/blob/master/design/15292-generics.md), by Ian Lance Taylor January 2011 - April 2016
    - [Type functions](https://github.com/golang/proposal/blob/master/design/15292/2010-06-type-functions.md), June 2010
    - [Generalized types](https://github.com/golang/proposal/blob/master/design/15292/2011-03-gen.md), March 2011
    - [Generalized types](https://github.com/golang/proposal/blob/master/design/15292/2013-10-gen.md), October 2013
    - [Type parameters](https://github.com/golang/proposal/blob/master/design/15292/2013-12-type-params.md), December 2013
    - [Compile-time Functions and First Class Types](https://github.com/golang/proposal/blob/master/design/15292/2016-09-compile-time-functions.md), September 2016
 - [Generics — Problem Overview](https://github.com/golang/proposal/blob/master/design/go2draft-generics-overview.md), by Russ Cox August 2018

This list of proposals reminded me of another story of highly requested feature - dependency management. Just briefly scratching the surface and reading the last proposal stunned me with how similar it's to [the sad story of go dep](https://peter.bourgon.org/blog/2018/07/27/a-response-about-dep-and-vgo.html). The story goes like this: in the response of community call to have a package management in go "a single man" went over the heads of the community and created gomod. Don't get me wrong, gomod as a dependency management system turned out to be great, but even the man himself (famous go tech lead) called it nothing else but [community mishandling](https://twitter.com/_rsc/status/1022588240501661696?lang=en).

In my eyes, generics story developed akin (maybe I'm overstretching here but I see at least some analogies between those two stories). The core go team had some internal particular, may be somewhat changing, vision of meta programming in go for years. And when time came, well, they went to implement it - `veni, vidi, vici`. It's worth a little that the big chunk of community was successfully adopting the code generation that de facto was considered as the main meta programming approach prior to generics. In result of this internal vision we got [contract generics draft](https://github.com/golang/proposal/blob/master/design/go2draft-contracts.md), which left community mostly disappointed. However, unlike situations with dependency management, this time go team acted smarter. Instead of forcing community, they hired a known and respected 3-rd party to fix the generic design, see [Featherweight Go](https://www.youtube.com/watch?v=Dq0WFigax_c) by Phil Wadler. This moved a part of responsibility for go generics design out of the core go team. In my eyes, this kind of move defused the situation around generics, and made the community mostly happy with generics design in go. Still, in my eyes, It didn't resolve core issues with generics in go.

## The problems with generics in go

Now, as we briefly walked through the most significant go generics milestones as I see and understand them. Let me highlight the most important points - why I think that generics is a bad idea how to introduce a meta programming to go.

#### 1. Generics make existing go type system unsound.

Go has "limited generics" inside already, even [FAQ](https://golang.org/doc/faq#generics) says so "meanwhile, Go's built-in maps and slices". Introducing new form of generics that is orthogonal to those types will make existing built-in maps, slices, channels to not fit. Soon enough "better performing"  generic vectors, maps, queues, etc. libraries will spring up like mushrooms. Which inevitably leads to flaking std library (welcome to C++), where you can find tens of different popular string type implementations.

#### 2. Generics use case is narrow, but they add a lot of weight to the language.

Well, we know why [generics are useful](https://blog.golang.org/why-generics). My question here is: does generics usefulness overweights the cost for go or not? The thing, as I see it from my experience, go doesn't really require generic most of the time. Again, referring to the latest [survey 2020](https://blog.golang.org/survey2020-results).

![survey 2020 results](/survey_2020_usage.png)

90th percentile of all applications written in go, accordingly to the survey, is either server side API/RPC code or data processing pipelines or some tooling around them. For all of these domains, I personally, find polymorphic programming in general and generics in particular rarely useful. Going forward, most of the go projects are designed to be small and use service based approach. Sure, those smaller projects could benefit from generics too and they already do, by using built-in maps, slices and channels. What they don't need almost 100% of the time - user defined generics, as it brings unnecessary complexity and bloats their small and simple code. But what about those bigger projects written in go, you might ask. I think, the answer is - golang shouldn't even try to match those "big projects" needs and period. Because, in my opinion, it's impossible to match `n` radically different requirements for `n` projects which size is comparable to go project itself.

#### 3. Generics are impossible to take back from the language.

Generics is a huge change, maybe the biggest change that will be ever introduced to go (from cognitive perspective at the very least). It worth to take a step back and say that go doesn't really follow [semver spec](https://semver.org/) closely for versioning their releases. The second part of go release version represents `major` version instead of semver mandated backwards `minor` version. So an idea of releasing generics beta under `1.18+` is fine, technically. But I personally will argue that generics has to be in go2.x if in any version at all. Mainly because of two related reasons:
- generics impact, effectively generics will create new sub language inside go (as it already had  happened with all languages that went this very path before) and will have various implications on std library, existing code bases, new libraries development, etc.
- mudding existing go1.x with a feature that never can be removed and which splits the language on before generics and after generics.

With that said, I think adding generics as-is or redesigning them to accomplish more into go2.x and not into go1.x might be a good middle ground. At the very least, it will buy some more time to think and sleep on it, which is a really goish way of doing things.

#### 4. Generics will make a part of comunity disappointed.

After years and years of go community defeneding absence of generics and learning how to live with tradeoffs of language design simplicity to language features. We, suddenly, got 180 degree turn in 1 year time window (I bet, nobody treated generics proposals as something feasible before that). Changing people mentality and beliefs is not a simple process, and I'd imagine a lot of people got frustrated because of this turn. Subjectively, despite of survey results, I think, that the true majority of people working with go on a daily basis didn't want generics.

#### 5. Generics implementation matters a lot, where generics via "boxing" doesn't bring a lot of value in go.

There is a [brilliant article](https://thume.ca/2019/07/14/a-tour-of-metaprogramming-models-for-generics/) that describes models of generics and metaprogramming and their implementation in various languages in depth. I don't want to dive too deep into comparison of two major classes of solutions to generics: "boxing" and "monomorphization" here in this article. I just say that if go team will decide to use "boxing" for the generics implementation, it will be a terrible misttake. In result, technically, just adding a syntax sugar on top of beloved `interface{}` type, thus trading language complexity for a syntax sugar.

## Alternatives

Summarizing all said above, I think that go generics is a bad idea, at least for go1.x, in any imaginable form. There are some alternative options that could facilitate the need for meta programming in go2.x that I feel might be more natural for go, from the least to the most invasive:
- keep the language it as-is, go was doing great in keeping generics status quo for years, why should it ever change if the absolute majority of use cases have no profit from them;
- add a few more generic built-in types (and functions) similarly to existing maps, slices, channels type inside the language runtime, I'd imagine couple basic popular data structure types like: list, set and ordered map will make vast majority of the people, asking for generics in go, satisfied;
- embrace and improve code generation and macrosing tooling as comunity driven solutions;
- accept generics in go2.x but also make existing type system maps, slices, channels work well with it (will most likely require to break existing type system completely), also, perhaps, it will demand to add operators overloading as well (to entitle user defined generics to be similar to built-in types), I'd expect final product to have a C++ style;
- dive much deeper into meta programming and prefer far superior compile time execution instead, e.g. [zig](https://ziglang.org/documentation/master/#comptime) does this, in fact you can even find official proposal in the past [Compile-time Functions and First Class Types](https://github.com/golang/proposal/blob/master/design/15292/2016-09-compile-time-functions.md) that, sadly, didn't get a lot of attention;

In conclusion, I shared my personal thoughts on go generics journey and why I'm dissatisfied with the current state of the generics in go. I just tried to express my thoughts and hopefully didn't hurt anyone's feelings, and If I did, please take my apology in advance. All and all, adding a meta programming support in a quite popular language with a long history of battling generics - is not an easy task. Generics were and still are the long requested feature by community, and their addition to the language might be good in the very end, but currently I have mixed feelings about them. My only hope is that some weak points of current generics desing will not make the way to the final public go release. Thank you for reading and as always stay tuned for more reading! 
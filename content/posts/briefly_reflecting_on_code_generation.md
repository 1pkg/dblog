---
author: 'Kostiantyn Masliuk'
title: 'Briefly reflecting on code generation'
date: '2022-01-19'
---

Not so long ago I've been working on Go CLI meta-tool [gofire](https://github.com/1pkg/gofire) that actively uses code generation. The tool was designed to automatically generate a command line interface for Go functions and do all required plumbing in between. The tool was inspired by [python-fire](https://github.com/google/python-fire).

I consciously chose code generation over any other alternatives because of guarantees of simplicity, predictability and performance. And while I like the results I was able to achieve using it for this project, yet I hit a dozen of common pitfals on my way.

First and foremost, code generation indeed delivers great predictability to end users. It fundamentally grants better dependencies control over the code, by making generated code to be a part of your project directly and explicitly. Thus moving ownership of the code a bit closer to you. Which makes it easier to read and modify it if needed.

However, simplicity and performance parts are not coming along with each other that well. It's hard enough to write clean and performant code on its own, but generating clean and performant code is exponentially harder. Because of indirect nature of code generation and lack of good standard tools for writing code that generates code, you more often than not will appear in a situation of stitching together several string buffers containing generated code to then try to quickly compile and simulate it somehow. Which often leads to situations when some corners are getting cut especially if certain flexibility is required in generation process. Therefore trading clean and performant code for simplicity of generation.

For sure Go is a great language to be code generated. The language is super explicit, it strives for simple semantics and syntax and it usually has the only one way of doing anything inside which are all handy properties for code generation. Go also provides built in [packages](https://pkg.go.dev/golang.org/x/tools/imports) to format generated code in a standard way. However unfortunately, it's where codegen story ends in Go. There is no clear way defined in Go standard library to support code generation to this day. In the wild in most cases you'll find just strings manipulations used, in other cases you'll see usage of [templates](https://pkg.go.dev/text/template), but usually that's about it. These approaches to codegen are less than ideal and make the process unnecessary painful. There are some libraries outside of stdlib that could somewhat alleviate this pain, like [jennifer](https://github.com/dave/jennifer).

To summarize, code generation is an old valuable technique that is easy to learn but hard to master. Which usually undeservedly doesn't get enough attention, but along with that it is a great tool in certain situations.

---
author: 'Kostiantyn Masliuk'
title: 'One more aspect of struct layout in Go'
date: '2021-12-24'
---

Some days ago I got a new intriguing [issue](https://github.com/1pkg/gopium/issues/24) opened for [gopium](https://github.com/1pkg/gopium) which is one of my projects, I maintain. In a few words: `fieldalign` linter in govet complained about unoptimal fields with pointer data ordering produced by [gopium](https://github.com/1pkg/gopium) memory packing strategy.

On a surface, the issue looks strange. Why would ordering of fields containing pointers inside structure matter? Many developers know for the fact that structure alignment and field order matters in a number of situations. It dictates how compiled program will allocate and interact with underlying memory beneath that structure. And while it's widely known that certain fields arrangement can eliminate unnecessary memory padding inside structures, and certain special extra bytes padding aligned to the cache size can prevent false sharing (for more info, refer to [this paper](https://www.usenix.org/legacy/publications/library/proceedings/als00/2000papers/papers/full_papers/sears/sears_html/index.html)). You will hardly find any descriptive information of why it's important to put fields with pointer data at the structure's beginning. In fact, as it turns out it's not the universal rule. For languages like: C or C++ or Rust, there should be no difference in such fields arrangement. Nevertheless it matters a lot for Go, and here is why - In the mark phase of Go garbage collector, when objects inside memory are scanned, Go uses a simple heuristic to avoid full structure scan by scanning only up to the very last pointer field inside this structure.

Take a closer look at [`mgcmark scanobject`](https://go.dev/src/runtime/mgcmark.go) implementation. The maximum struct scan offset is determined using next code snippet, where each field is checked against bit `scan` mask; if the current scanned field doesn't match the mask i.e. doesn't have pointer data inside - scan is finished immediately.

```go
for i = 0; i < n; i, hbits = i+sys.PtrSize, hbits.next() {
    // Load bits once. See CL 22712 and issue 16973 for discussion.
    bits := hbits.bits()
    if bits&bitScan == 0 {
        break // no more pointers in this object
    }
    if bits&bitPointer == 0 {
        continue // not a pointer
    }
    // ...
}
```

Knowing all of this, we can draw the conclusion that it's a quite important to keep pointer containing fields at the beginning of structs in Go to reduce GC CPU time spent on objects scan. So I went ahead and patched [gopium](https://github.com/1pkg/gopium) memory packing strategy to match [`fieldalign`](https://cs.opensource.google/go/x/tools/+/refs/tags/v0.1.7:go/analysis/passes/fieldalignment/fieldalignment.go;l=145).

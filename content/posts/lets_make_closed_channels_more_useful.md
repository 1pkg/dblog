---
author: 'Kostiantyn Masliuk'
title: "Let's make closed channels more useful"
date: '2021-01-02'
---

## Go zero values

Zero values is an extremely useful feature of golang, you can read [nice detailed article](https://dave.cheney.net/2013/01/19/what-is-the-zero-value-and-why-is-it-useful) about why they are so important. They are not only making day to day programming simpler but also save everyone from whole layer of "uninitialized variable" errors specific to languages like C. Personally I think that it would be beneficial if go could provide us with more useful safe zero value for all builtin structures. If you don't know, there are some unsafe builtin zero `nil` value structures in go; namely most problematic ones are maps. But channels and slices could benefit from safe zero values as well. I'm not alone in my thoughts here, see [one of such proposals](https://github.com/golang/go/issues/28133) of adding safer zero values to all builtin types. It's pity that this was not foreseen originally, but I think now these changes can't be introduced to runtime directly as they will break enormous amount of existing gocode and on top of that make existing code less efficient.

Recently while I was working on [`gotcha`](https://github.com/1pkg/gotcha) - library for tracing allocated memory per goroutine, see [my previous post](/posts/lets_trace_goroutine_allocated_memory/). I figured that same "patching" technique could be also applied to a wide range of runtime modifications, e.g. we could try to change maps zero `nil` value to safer alternative or go even further and change some core operations semantic.

## Channels close semantic

Channels are the single most important concurrent primitives in go. They are directly or indirectly used in a wide variety of tasks for any imaginable async communication between goroutines. They are first class citizen structures in go and go provides a special set of operations for them, like separate: `send ->` and `recv <-` operators, powerful `select` statement, builtin buffering, etc. Channels are great, but as everything inside go, they follow "simple by default" convection which strictly demands "the single way" of doing anything. It's great rule usually, but sometimes it might be a bit inconvenient - for example when any given channel is closed, in receivers after receiving all buffered values from the channel all further "closed" values received from this channel will be empty zero values!

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 2)
    ch <- 100
    close(ch)
    v, ok := <- ch
    fmt.Println(v, ok) // 100, true
    v, ok = <- ch
    fmt.Println(v, ok) // 0, false
    v, ok = <- ch
    fmt.Println(v, ok) // 0, false
    v, ok = <- ch
    fmt.Println(v, ok) // 0, false
}
```

Using default type zero value as "closed" channel value, again, is perfectly logical and valid approach and in most real world scenarios it works great. But sometimes an extra flexibility could simplify channel interactions. One of essential communication patterns - "fanout" often is implemented via channel close call in go. This way it's easy to make a "fanout" signal for receivers but what is not that simple to deliver a message along with that signal, yet it's simple enough to achieve the same effect with storing synchronized messages separately, although not that convenient. In my eyes, it might be an interesting idea to have a separate version of `close` called `close2` in go with semantic close to this.

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 2)
    ch <- 100
    close2(ch, 200)
    v, ok := <- ch
    fmt.Println(v, ok) // 100, true
    v, ok = <- ch
    fmt.Println(v, ok) // 200, false
    v, ok = <- ch
    fmt.Println(v, ok) // 200, false
    v, ok = <- ch
    fmt.Println(v, ok) // 200, false
}
```

I also understand that adding such specific tool inside go standard library is overkill and will lead only to unnecessary confusions, so separate library probably is an ideal place for such function.

## Introduction of golatch library

Meet [golatch](https://github.com/1pkg/golatch) that seamlessly patches go runtime to provide a way to close a chan idempotently + overwrite empty value returned from that closed channel.

<table>
<tr>
<th>With Golatch</th>
<th></th>
<th>Without Golatch</th>
</tr>

<tr>
<td>

```go
package main

import (
	"fmt"
    "sync"

	"github.com/1pkg/golatch"
)

func main() {
	ch := make(chan int)
	var wg sync.WaitGroup
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go worker(i, ch, &wg)
	}
	golatch.Close(ch, 10)
	wg.Wait()
}

func worker(i int, ch chan int, wg *sync.WaitGroup) {
	v, ok := <- ch // 10, false
	if ok || v != 10 {
		panic("unreachable") // won't panic
	}
	fmt.Printf("worker %d chan is closed with value %d\n", i, v)
	wg.Done()
}
```

</td>
<td></td>
<td>

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	ch := make(chan int)
	var wg sync.WaitGroup
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go worker(i, ch, &wg)
	}
	close(ch)
	wg.Wait()
}

func worker(i int, ch chan int, wg *sync.WaitGroup) {
	v, ok := <- ch // 0, false
	if ok || v != 10 {
		panic("unreachable") // will panic
	}
	fmt.Printf("worker %d chan is closed with value %d\n", i, v)
	wg.Done()
}


```

</td>
</tr>
</table>

Golatch exposes single function `Close` that idempotently closes any provided channel and stores "closed" value for any further reading from that channel. When "closed" channel read occures default chan type zero value will be overwritten by previously stored "closed" value. Note that next calls to `Close` on the same channel will overwrite stored value with provided new value. Note also that `Close` returns `Cancel` eviction function that should be called in order to remove stored value and restore default receive behavior on the channel.

## How does golatch even work?

In order to understand what we need to do, we need to understand how go runtime handles channel receive and close in the first place. After some reading inside go runtime we can find next definitions.

```go
// entry points for <- c from compiled code
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}
```

```go
//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

//go:linkname reflect_chanrecv reflect.chanrecv
func reflect_chanrecv(c *hchan, nb bool, elem unsafe.Pointer) (selected bool, received bool) {
	return chanrecv(c, elem, !nb)
}
```

Those are chan receive entrypoint operations that we were searching for. Now the work is trivial - we just need to reflink and patch them, see my [previous article](/posts/lets_trace_goroutine_allocated_memory/) for more info how this could be done. The basic idea of patched method is: reuse runtime helper `chanrecv` first, then check whether channel is closed or not, if channel is closed check whether any previous value for this channel was stored, if value is found in storage - overwrite default chan receive value and return this value instead. Luckily all those receive entrypoint functions are simply using `chanrecv` helper underneath so no extra patching effort unlike in [gotcha](https://github.com/1pkg/gotcha) is required.

```go
gomonkey.Patch(chanrecv1, (ch *hchan, elem unsafe.Pointer) {
    chanrecv(ch, elem, true)
    // apply any action only if chan is closed
    if ch.closed {
        // if the store has relevant "close" value
        if store.has(ch) {
            // get such value from the store
            v := store.get(ch)
            // and set it to default return value
            set(elem, v)
        }
    }
})
```

As a mindful reader can notice we made just a half of job by patching `chanrecv` entrypoint methods. Indeed all receiving methods are done and any `<-` expression will work as expected and return proper results, but what about `select` statement? After reading more code inside go runtime we can find next definitions for simple `select` cases.

```go
// compiler implements
//
//	select {
//	case v = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if selectnbrecv(&v, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}

// compiler implements
//
//	select {
//	case v, ok = <-c:
//		... foo
//	default:
//		... bar
//	}
//
// as
//
//	if c != nil && selectnbrecv2(&v, &ok, c) {
//		... foo
//	} else {
//		... bar
//	}
//
func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
	// TODO(khr): just return 2 values from this function, now that it is in Go.
	selected, *received = chanrecv(c, elem, false)
	return
}
```

And generic select entrypoint method as the cherry 🍒 on a pie 🥧.
I highly recommend you to read an additional [excellent article](https://programming.vip/docs/deeply-understanding-the-principles-of-go-channel-and-select.html) on how channel operations work if you wanna understand `select` statement in depth.

```go
// selectgo implements the select statement.
//
// cas0 points to an array of type [ncases]scase, and order0 points to
// an array of type [2*ncases]uint16 where ncases must be <= 65536.
// Both reside on the goroutine's stack (regardless of any escaping in
// selectgo).
//
// selectgo returns the index of the chosen scase, which matches the
// ordinal position of its respective select{recv,send,default} call.
// Also, if the chosen scase was a receive operation, it reports whether
// a value was received.
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool)
```

Now we need to patch all those methods; patching `selectnbrecv` and `selectnbrecv2` is quite an easy task, `selectgo` is on the other hand is much trickier. The simplest way to pass by - to use patch guard discussed in my [previous post](/posts/lets_trace_goroutine_allocated_memory/). One gotcha though, patch guard is unsafe to use; it's not thread safe - while function code is unpatched back to original `selectgo` another `selectgo` call could happen in between, resulting in reading chan type zero value. This race could be quite awful, however I feel that for golatch use case patch guard advantages cleatly outweigh disadvantages and patch guard could be used as simplest solution. In anyway patching is generally a terrible unsafe idea and should NOT be used in any real code by anyone.

## Final conclusion

As with my [previous article](/posts/lets_trace_goroutine_allocated_memory/) we can highlight that any thinkable language design change could be achieved (even change of core go semantic) if you dig deeply enough and agree to use some extreme techniques like [go linkname](/posts/lets_trace_goroutine_allocated_memory/#unexported-function---golinkname-to-rescue) and [code patching](/posts/lets_trace_goroutine_allocated_memory/#allocation-hooking---code-patching-to-rescue). On the other hand while doing that, you should be constantly asking yourself - "is it really worth doing so in a such wicked way?". And I will argue that in most cases the answer is "no".

As I said previously the greatest merit of go “features and implementations conservatism” should always prevail; don't fight your language instead embrace it. Nevertheless if you know what you are doing sometimes you can bend the rules and experiment even with the core concepts.

In my final conclusion I highly discourage anyone to use golatch in any production code or maybe even any code at all as it’s extremely unsafe and not reliable and won’t be supported in foreseeable future. Nevertheless, one could probably imagine reasonable use cases for this concept library in go benchmarks or test. Finally it's worth to say that primary intention to develop golatch was learning and that golatch doesn't have comprehensive tests coverage and support and not ready for any serious use case anyway.

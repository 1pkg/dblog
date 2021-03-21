---
author: 'Kostiantyn Masliuk'
title: 'Writing complex concurrent table driven tests'
date: '2021-03-21'
---

Recently, I've finished my reading of "Software Engineering at Google" book. Despite the fact that this book contains considerable amount of: information duplicity, obvious advices and quite generic explanations. I still can suggest this book to anyone interested in understanding the modern Software Engineering industry. Although be advised that book doesn't go very deep with any given topic. Rather, it's just quite nicely composed collection of easy to read and understand thoughts from Google engineers.

One subject highlighted in this book got me thinking more than others - testing. The book spends quite some time on it, having sections: "Testing Overview", "Unit Testing", "Test Doubles", "Larger Testing", etc. And while most of the points given by the authors here are 100% undeniable:

- "follow proper ratio between unit/integration/e2e tests 80/15/5 (also known as test pyramid)";
- "test everything that is important for your application, i.e. anything that brings you the value directly or indirectly (so-called Beyoncé rule)";
- "do not over use mocks, prefer to use real implementation instead (test behavior not methods)";
- "try to avoid having brittle tests as much as you can (in the long run they might appear to be a big problem)";
- "aim for simplest possible test, try to not use any logic inside your tests";
- and so much more other useful and practical advices;

It's definitely worth to say that all of these advices are not ultimate truth and specific needs of specific systems could outweigh them easily.

Not so long ago, I was adding quite complex test suites for one of my open source projects throttler library [gohalt](https://github.com/1pkg/gohalt). Originally, I underestimated this task, thinking: "how hard it would be to add some decent test coverage to project that is already well prepared to be tested (it has one core throttler interface and 30+ throttlers implementations) ... it should be super easy to use test table here, right?". I was horribly wrong 🤪. For the beginning, we need to understand that logically all throttlers are used in concurrent muli-threading environments (otherwise, why would you use throttlers in the first place). Therefore, tests for thottlers require to have heavy concurrent testing workload; otherwise they provide test scenarios far from reallity and generally have value close to zero. But one does not simply - to have one test table to rule all throttlers that act differently and need radically different setup for concurrent workload.

What does "Software Engineering at Google" book suggest to do in such case scenario? I believe majority of readers will answer the rule still stands "do not add any complex logic inside you test". But then we have only single choice here: ditch out nice table driven testing approach and use separate test case scenario for each trottler with separate setup helper for each test case respectively. Although this approach is sound on paper, for `gohalt` it has single fatal flow - the amount of testing boilerplates will be outstading. It's hard to say with certainty how much testing code will be needed. But my expectations are that amout of testing code will be at least double amount of actual project code itself 😮 maybe even more. While it might be acceptable and even preferable to someone to have huge, verbose but very explicit test suite, I decided to cheat a bit and go smart on this instead. I went with using table driven testing approach with separate "overcomplicated" test case setup structure for any given test case [full gohalt test code](https://github.com/1pkg/gohalt/blob/master/throttlers_test.go).

Let's look on resulting code briefly.

```go
var trun Runner = NewRunnerSync(context.Background(), NewThrottlerBuffered(1))

type tcase struct {
	tms  uint64            // number of sub runs inside one case
	thr  Throttler         // throttler itself
	acts []Runnable        // actions that need to be throttled
	pres []Runnable        // actions that neeed to be run before throttle
	tss  []time.Duration   // timestamps that needs to be applied to contexts set
	ctxs []context.Context // contexts set for throttling
	errs []error           // expected throttler errors
	durs []time.Duration   // expected throttler durations
	idx  uint64            // carries seq number of sub run execution
	over bool              // if throttler needs to be over released
	pass bool              // if throttler doesn't need to be released
	wait time.Duration     // if next throttler run needs to be delayed in acq
}

func (t *tcase) run(index int) (dur time.Duration, err error) {
	// get context with fallback
	ctx := context.Background()
	if index < len(t.ctxs) {
		ctx = t.ctxs[index]
	}
	// run additional pre action only if present
	if index < len(t.pres) {
		if pre := t.pres[index]; pre != nil {
			_ = pre(ctx)
		}
	}
	var ts time.Time
	// try catch panic into error
	func() {
		defer atomicIncr(&t.idx)
		defer func() {
			if msg := recover(); msg != nil {
				err = msg.(ErrorInternal)
			}
		}()
		ts = time.Now()
		// force strict acquire order
		for index != int(atomicGet(&t.idx)) {
			_ = sleep(ctx, time.Microsecond)
		}
		// set additional timestamp only if present
		if index < len(t.tss) {
			ctx = WithTimestamp(ctx, time.Now().Add(t.tss[index]))
		}
		err = t.thr.Acquire(ctx)
		// in case of threshold error
		if terr, ok := err.(ErrorThreshold); ok {
			// check whether it's durations threshold
			if durs, ok := terr.Threshold.(strdurations); ok {
				// then round current values to milliseconds
				durs.current = durs.current.Round(time.Millisecond)
				terr.Threshold = durs
				err = terr
			}
		}
		// if next throttler run needs to be delayed sleep for wait
		if t.wait > 0 {
			_ = sleep(ctx, t.wait)
		}
	}()
	dur = time.Since(ts)
	// run additional action only if present
	if index < len(t.acts) {
		if act := t.acts[index]; act != nil {
			_ = act(ctx)
		}
	}
	limit := 1
	if t.over && uint64(index+1) == t.tms { // imitate over releasing on last call
		limit = index + 1
	}
	if t.pass {
		limit = 0
	}
	for i := 0; i < limit; i++ {
		if err := t.thr.Release(ctx); err != nil {
			return dur, err
		}
	}
	return
}

func (t *tcase) result(index int) (dur time.Duration, err error) {
	if index < len(t.errs) {
		err = t.errs[index]
	}
	if index < len(t.durs) {
		dur = t.durs[index]
	}
	return
}
```

Structure `tcase` represents single test case data set for any given throttler. It flexibly configures what actions should be applied to the throttler, how many times, in which order, with what extra options and what results should we expect after this. The main idea here - for each test case to define how many parallel concurrent calls needs to be applied to the throttler `tms`; to define ordered lists of actions `acts` and `pres` to be executed on the throttler; to define ordered lists of options that need to be propagated to the throttler `tss`, `ctxs`; define global options for the run `over`, `pass`, `wait`; and finally to define ordered lists of result expectations `errs`, `durs`. To ensure strict ordering, simple spin lock like technique is used.

```go
func() {
		defer atomicIncr(&t.idx)
		// force strict acquire order
		for index != int(atomicGet(&t.idx)) {
			_ = sleep(ctx, time.Microsecond)
		}
		err = t.thr.Acquire(ctx)
	}()
```

The heart of test routine itself is placed inside main test entry point loop.

```go
func TestThrottlers(t *testing.T) {
	table := map[string]tcase{
		// code here is omitted
	}
	for tname, ptrtcase := range table {
		t.Run(tname, func(t *testing.T) {
			var wg sync.WaitGroup
			wg.Add(int(ptrtcase.tms))
			for i := 0; i < int(ptrtcase.tms); i++ {
				t.Run(fmt.Sprintf("run %d", i+1), func(t *testing.T) {
					go func(index int, tcase *tcase) {
						defer wg.Done()
						rdur, rerr := tcase.result(index)
						dur, err := tcase.run(index)
						trun.Run(func(context.Context) error {
							log("expected error %v actual err %v", rerr, err)
							log("expected duration le %s actual duration %s", rdur/2, dur)
							require.Equal(t, rerr, err)
							require.LessOrEqual(t, int64(rdur/2), int64(dur))
							return nil
						})
					}(i, &ptrtcase)
				})
			}
			wg.Wait()
		})
	}
}
```

Nothing super fancy happens here; we iterate over test cases table and run them one by one. Note two things though: first we cannot use t.Parallel runner here as we need strict control over execution ordering for a sinlgle test case and second we are using approximate comparison of resulting durations `require.LessOrEqual(t, int64(rdur/2), int64(dur))` to reduce test brittleness.

I acknowledge that this code is far from masterpiece. Moreover, it even fails occasionally (from my experience false positive rate is less than couple percents) which could be resolved by further adjusting comparison durations threshold and tuning individual test cases. But overal, this code works great and it helped me to reduce test helper boilerplates to the bare minimum.

Instead of any conclusion, I want to provide some numbers for [gohalt](https://github.com/1pkg/gohalt) tests. The ratio of test setup helper code to real test case scenarios is approximately `1:7`. The ratio of test code to library code itself is an approximately `1:1` while vast majority of all code is fully covered. I highly doubt that the same amount of tests with the same coverage could be possible if the rule of simple tests was followed. So, as always, don't follow any Software Engineering dogma blindly, think what suits your situation best and stay tuned for more reading!

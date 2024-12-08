---
title: "Coming in Go 1.24: testing/synctest experiment for time and concurrency testing"
date: 2024-12-08T09:01:58-04:00
---

Testing code that involves time or concurrency can be a struggle.
It often leads to hard-to-debug flakes in CI or long-running tests.

Go 1.24 is [scheduled to be released](https://go.dev/wiki/Go-Release-Cycle) in February and the release freeze has begun.

It's set to include an [experimental `testing/synctest` package](https://github.com/golang/go/issues/69687)
designed to make testing code that involves time or concurrency precise and fast.

I'm pretty excited about it!

## Time testing trouble

Suppose you have a test that looks like this:

``` go
func Test(t *testing.T) {
    before := time.Now()
    time.Sleep(time.Second)
    after := time.Now()
    if d := after.Sub(before); d != time.Second {
        t.Fatalf("took %v", d)
    }
}
```

You'll typically run into two issues.

First, it's pretty hard to get it to pass:

``` shell
> go test .
=== RUN   Test
    x_test.go:25: took 1.00106725s
--- FAIL: Test (1.00s)
FAIL
```

Second, it actually takes a second to run.
In more complete scenarios or with more tests that time can add up.

To get it passing more consistently, you could change to something like:

``` go {hl_lines=[5]}
func Test(t *testing.T) {
    before := time.Now()
    time.Sleep(time.Second)
    after := time.Now()
    if d := after.Sub(before); d > 2*time.Second {
        t.Fatalf("took %v", d)
    }
}
```

That might work locally or in some environments.
It's likely to be flaky in slower or more contended setups, such as CI.

It still actually takes a second to run, though.

## How synctest helps

The `synctest.Run` function runs the function given to it in a "bubble."
Time inside the bubble is controlled by the Go runtime and only advances when all goroutines are idle.
Goroutines are considered idle when calling `time.Sleep` or receiving on a channel, for example.
Time can advance instantly instead of having to wait for real time to pass.

If we change our test (and imports above) to:

``` go
import (
	"testing"
	"testing/synctest"
	"time"
)

func Test(t *testing.T) {
	synctest.Run(func() {
		before := time.Now()
		time.Sleep(time.Second)
		after := time.Now()
		if d := after.Sub(before); d != time.Second {
			t.Fatalf("took %v", d)
		}
	})
}
```

And then use [gotip](https://pkg.go.dev/golang.org/dl/gotip) with `GOEXPERIMENT=synctest`, we get:

``` shell
> GOEXPERIMENT=synctest gotip test -v
=== RUN   Test
--- PASS: Test (0.00s)
PASS
```

There are two things to note.

First, it passed! Second, it took ~zero seconds instead of one.

Logging the `before` and `after` values shows the controlled time advancement in action:

``` text
    x_test.go:17: before: 2000-01-01 00:00:00 +0000 UTC ...
    x_test.go:18: after: 2000-01-01 00:00:01 +0000 UTC ...
```

## Extending to concurrency

Suppose we have a test like this:

``` go
func Test(t *testing.T) {
	ctx := context.Background()

	ctx, cancel := context.WithCancel(ctx)

	var hits atomic.Int32
	go func() {
		tick := time.NewTicker(time.Millisecond)
		defer tick.Stop()
		for {
			select {
			case <-ctx.Done():
				return
			case <-tick.C:
				hits.Add(1)
			}
		}
	}()

	time.Sleep(3 * time.Millisecond)
	cancel()

	got := int(hits.Load())
	if want := 3; got != want {
		t.Fatalf("got %v, want %v", got, want)
	}
}
```

This passes locally but it's flaky.
Even though it does pass most of the time, it has a subtle bug around the Ticker's initial delay.

``` text
> go test -count 1000 -failfast
--- FAIL: Test (0.00s)
    x_test.go:47: got 2, want 3
FAIL
```

synctest can help with this, too.

First, we wrap it in `synctest.Run`:

``` go
func Test(t *testing.T) {
	synctest.Run(func() {
		ctx := context.Background()

		ctx, cancel := context.WithCancel(ctx)

		var hits atomic.Int32
		go func() {
			tick := time.NewTicker(time.Millisecond)
			defer tick.Stop()
			for {
				select {
				case <-ctx.Done():
					return
				case <-tick.C:
					hits.Add(1)
				}
			}
		}()

		time.Sleep(3 * time.Millisecond)
		cancel()

		got := int(hits.Load())
		if want := 3; got != want {
			t.Fatalf("got %v, want %v", got, want)
		}
	})
}
```

Then we see our bug:

``` text
> GOEXPERIMENT=synctest gotip test -v
=== RUN   Test
    x_test.go:49: got 2, want 3
--- FAIL: Test (0.00s)
FAIL
```

Since the Ticker has an initial delay, we need to sleep 4 ms to get 3 hits.
Once that's fixed:

``` text
> GOEXPERIMENT=synctest gotip test -v
=== RUN   Test
--- PASS: Test (0.00s)
```

## Conclusion

It seems that `testing/synctest` will significantly improve testing code that involves time or concurrency.
At work, I've already tried it on some flaky tests like the ones above and it's helped.

You can try it yourself now by using `gotip` and setting `GOEXPERIMENT=synctest`.
When Go 1.24 comes out `GOEXPERIMENT=synctest` will still be required.

Review the [main proposal](https://go.dev/issue/67434) and share any experience you have.

There are also some encouraging examples in the wild, all by Damien Neil:
* [An etcd test is changed to using synctest with success](https://github.com/golang/go/issues/67434#issuecomment-2430263934)
* [A net/http test that took 12s now takes 0s](https://go.dev/cl/630382)
* [A net/http test that was flaky and slow is now precise and fast](https://go.dev/cl/631795)

Thanks to Damien Neil for initiating this proposal and building out the implementation for us to try!

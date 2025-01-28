---
layout: post
title:  "Loop-invariant code motion"
---

While watching Anthony Shaw's PyCon US [talk](https://youtu.be/YY7yJHo0M5I?t=659) I came across "loop invariance" and the technic that compilers do called "loop-invariant code motion". As I mostly used interpreted language (Python) and only recently started to look into compiled one (Go), it was interesting to learn what compilers can do for optimization. For me it was obvious not to put redundant computations in a hot loop, but with "loop-invariant code motion" it's different, it doesn't affect performance and even improves readability.

I decided to compare timings using Python 3.10[^1] (CPython and PyPy) and Go 1.23.


```python
import timeit


def inside_loop():
    a = [1, 2, 3, 4, 5] * 100
    for _ in range(1000):
        len(a)


def outside_loop():
    a = [1, 2, 3, 4, 5] * 100
    a_len = len(a)
    for _ in range(1000):
        a_len


if __name__ == "__main__":
    t1 = timeit.timeit("inside_loop()", number=1_000_000, globals=globals())
    t2 = timeit.timeit("outside_loop()", number=1_000_000, globals=globals())

    print(f"len() inside loop: {t1:.2f}s")
    print(f"len() outside the loop: {t2:.2f}s")
    print(f"outside_loop() is faster than inside_loop() by a factor of {t1 / t2:.2f}")

```

Running with CPython

```
$ uv run --python cpython-3.10 ./python/loop_invar.py 

len() inside loop: 54.34s
len() outside the loop: 22.28s
outside_loop() is faster than inside_loop() by a factor of 2.44
```

And with PyPy 

```
$ uv run --python pypy-3.10 ./python/loop_invar.py 

len() inside loop: 1.10s
len() outside the loop: 0.70s
outside_loop() is faster than inside_loop() by a factor of 1.57
```

So for CPython, it's ~144% faster, adn for PyPy, it's ~57% faster. But look at the absolute time, the PyPy is incredibly more performant. I'm not really sure that the "loop-invariant code motion" is actually applied in the PyPy implementation at all, as the difference is still quite substantial. So, I attribute the speedup due to general PyPy optimizations.

But now, let's look how Go compiler handles it.

```go
// main.go

package main

func insideLoop() {
	a := make([]int, 0, 500)
	for i := 0; i < 100; i++ {
		a = append(a, 1, 2, 3, 4, 5)
	}
	for i := 0; i < 100; i++ {
		_ = len(a)
	}
}

func outsideLoop() {
	a := make([]int, 0, 500)
	for i := 0; i < 100; i++ {
		a = append(a, 1, 2, 3, 4, 5)
	}
	aLen := len(a)
	for i := 0; i < 100; i++ {
		_ = aLen
	}
}
```

```go
// main_test.go

package main

import "testing"

func BenchmarkInsideLoop(b *testing.B) {
	for i := 0; i < b.N; i++ {
		insideLoop()
	}
}

func BenchmarkOutsideLoop(b *testing.B) {
	for i := 0; i < b.N; i++ {
		outsideLoop()
	}
}
```

Running the Go benchmark

```bash
$ go test -bench=. -benchtime=1000000x

goos: linux
goarch: amd64
pkg: loopinvariant
cpu: Intel(R) Core(TM) i7-8565U CPU @ 1.80GHz
BenchmarkInsideLoop-8            1000000               428.4 ns/op
BenchmarkOutsideLoop-8           1000000               438.8 ns/op
PASS
ok      loopinvariant   0.870s
```

We can see that the difference is much more subtle. OutsideLoop is faster than InsideLoop by a factor of (428.4ns / 438.8ns) = 0.98. InsideLoop is even faster, but let's attribute it to external factors and unknowns which I don't know.

On the plot below, we can see the difference between each implementation.

![Performance Comparison plot]({{ site.url }}/assets/images/2025-01-27-loop-invariant/performance_comparison.png)

For interpreted languages like Python, moving invariant computations outside the loop can have a significant impact on performance. But compiled languages like Go benefit from "loop-invariant code motion", resulting in no particular differences between the two implementations.

[^1]: Using 3.10, as this is the latest PyPy version at the moment.
---
title: "Microbenchmarking Go: Bytes vs. Runes and the Hidden Cost of Map Keys"
date: 2025-06-22T15:03:36+02:00
draft: false
toc: false
images:
tags: ["Go", "Benchmarking", "Performance", "Microoptimization"]
---

While working through the [Go track on Exercism.org](https://exercism.org/tracks/go), I ran into an interesting performance puzzle in the **Nucleotide Count** exercise. It looked simple at first glance, but it sent me down a rabbit hole of microbenchmarking that revealed an insight beyond just how you iterate over strings.

## The Initial Question: Bytes or Runes?

The input is a DNA sequence, which contain only ASCII characters like `A`, `C`, `G`, and `T`. Naturally, I reached for a `map` to count each nucleotide.

With that set up, I added the iteration over the string using byte indexing (`for i := 0; i < len(s); i++`), since I remembered a comment on [r/golang](https://www.reddit.com/r/golang/) suggesting that byte iteration is idiomatic and efficient for ASCII-only strings.

```go
package dna

import (
  "fmt"
)

type Histogram map[byte]int

func NewHistogram() Histogram {
  return Histogram{'A': 0, 'C': 0, 'G': 0, 'T': 0}
}

type DNA string

func (d DNA) Counts() (Histogram, error) {
  h := NewHistogram()

  for i := 0; i < len(d); i++ {
    if _, ok := h[d[i]]; !ok {
      return nil, fmt.Errorf("invalid nucleotide %q in DNA strand", d[i])
    }
    h[d[i]]++
  }

  return h, nil
}
```

## A Surprising Benchmark Result

One of my favorite parts of Exercism is reviewing other submissions and comparing microbenchmark results. This time, I noticed that a solution by `bobahop` actually [used rune iteration](https://exercism.org/tracks/go/exercises/nucleotide-count/approaches/switch-statement) (`for _, r := range s`) and surprisingly, outperformed mine in microbenchmarks.

That caught me off guard. Had I been misled by Reddit?

I assumed that rune iteration would be slower due to the UTF-8 decoding overhead. I also knew my version was slower than `bobahop`'s due to using `map` lookups, while his used a `switch`, so I tested his version with byte iteration instead. Even then, his rune-based solution remained faster, just as he had noted on Exercism.

```go
package dna

import "fmt"

type Histogram map[byte]int

type DNA string

const (
  nucA byte = 65
  nucC byte = 67
  nucG byte = 71
  nucT byte = 84
)

func (dna DNA) Counts() (Histogram, error) {
  results := Histogram{nucA: 0, nucC: 0, nucG: 0, nucT: 0}
  length := len(dna)
  for i := 0; i < length; i++ {
    nuc := dna[i]
    switch nuc {
    case nucA, nucC, nucG, nucT:
      results[nuc]++
    default:
      return nil, fmt.Errorf("invalid nucleotide '%c'", nuc)
    }
  }
  return results, nil
}
```

## My Initial Approach

To get to the bottom of this, I created a small repo to isolate and benchmark different implementations using both byte and rune iteration.

I started with a simpler function: transcribing DNA to RNA.

```go
func TranscribeDnaToRnaBytes(dna string, rna []byte) {
  for i := range len(dna) {
    switch dna[i] {
    case 'A':
      rna[i] = 'U'
    case 'C':
      rna[i] = 'G'
    case 'G':
      rna[i] = 'C'
    case 'T':
      rna[i] = 'A'
    }
  }
}

func TranscribeDnaToRnaRunes(dna string, rna []rune) {
  for i, n := range dna {
    switch n {
    case 'A':
      rna[i] = 'U'
    case 'C':
      rna[i] = 'G'
    case 'G':
      rna[i] = 'C'
    case 'T':
      rna[i] = 'A'
    }
  }
}
```

As expected, the benchmarks showed that **byte iteration was slightly faster** than rune iteration.

```sh
$ go test -bench='^(BenchmarkTranscribeDnaToRnaBytes|BenchmarkTranscribeDnaToRnaRunes)$' -benchtime=1000000x -benchmem
goos: linux
goarch: amd64
pkg: github.com/pessolato/strmapmicrobench
cpu: 12th Gen Intel(R) Core(TM) i9-12900F
BenchmarkTranscribeDnaToRnaBytes-24      1000000               331.4 ns/op             0 B/op          0 allocs/op
BenchmarkTranscribeDnaToRnaRunes-24      1000000               529.3 ns/op             0 B/op          0 allocs/op
```

Then I rewrote the nucleotide count functions, simplifying and updating them. I also used a more idiomatic byte loop (`for i := range s`) to see if that would help.

```go
func CountNucleotidesByByteIndex(dna string) map[byte]int {
  chars := make(map[byte]int)
  for i := range len(dna) {
    chars[dna[i]]++
  }
  return chars
}

func CountNucleotidesByRuneRange(dna string) map[rune]int {
  chars := make(map[rune]int)
  for _, r := range dna {
    chars[r]++
  }
  return chars
}
```

But the performance difference persisted.

```sh
$ go test -bench='^(BenchmarkCountNucleotidesByByteIndex|BenchmarkCountNucleotidesByRuneRange)$' -benchtime=1000000x -benchmem
goos: linux
goarch: amd64
pkg: github.com/pessolato/strmapmicrobench
cpu: 12th Gen Intel(R) Core(TM) i9-12900F
BenchmarkCountNucleotidesByByteIndex-24          1000000              5191 ns/op             192 B/op          2 allocs/op
BenchmarkCountNucleotidesByRuneRange-24          1000000              1682 ns/op             192 B/op          2 allocs/op
PASS
ok      github.com/pessolato/strmapmicrobench   6.876s
```

At this point, I realized the bottleneck might not be the loop. So I turned to profiling.

## CPU Profiling Tells the Real Story

I profiled both versions of the counting function and things started to click.

![Graph of CPU Profile](Profile-ByByteIndex-VS-ByRuneRange.png "Graph of CPU Profile")

The version using byte indexing was calling the generic `mapassign` function, while the rune-based version was using `mapassign_fast32`, which is much faster.

That was the aha moment. The real issue wasn’t string iteration, but **how maps work under the hood**, especially based on the **key type**.

## Go’s Internal Map Implementations

The Go runtime uses different internal implementations for `map` operations depending on the key type:

* For smaller key types like `byte`, it uses the slower generic `mapassign`.
* For `int32` (which `rune` is an alias of), it uses optimized versions like `mapassign_fast32`.

So, despite rune iteration having more overhead, the faster map operations made the overall function faster.

To test this theory further, I added two more functions: one that uses byte iteration but casts to `int16`, and another that casts to `rune`:

```go
func CountNucleotidesByByteIndexAsRune(dna string) map[rune]int {
  chars := make(map[rune]int)
  for i := range len(dna) {
    chars[rune(dna[i])]++
  }
  return chars
}

func CountNucleotidesByByteIndexAsInt16(dna string) map[int16]int {
  chars := make(map[int16]int)
  for i := range len(dna) {
    chars[int16(dna[i])]++
  }
  return chars
}
```

Benchmark results:

```sh
$ go test -bench='^BenchmarkCountNucleotidesBy.*$' -benchtime=1000000x -benchmem
goos: linux
goarch: amd64
pkg: github.com/pessolato/strmapmicrobench
cpu: 12th Gen Intel(R) Core(TM) i9-12900F
BenchmarkCountNucleotidesByByteIndex-24                  1000000              5177 ns/op             192 B/op          2 allocs/op
BenchmarkCountNucleotidesByByteIndexAsRune-24            1000000              1648 ns/op             192 B/op          2 allocs/op
BenchmarkCountNucleotidesByByteIndexAsInt16-24           1000000              5132 ns/op             192 B/op          2 allocs/op
BenchmarkCountNucleotidesByRuneRange-24                  1000000              1670 ns/op             192 B/op          2 allocs/op
PASS
ok      github.com/pessolato/strmapmicrobench   13.631s
```

Interestingly, `int16` didn’t get any runtime optimization. But casting bytes to `rune` actually performed better than the native rune iteration, because byte iteration is inherently faster, and `rune` still benefits from optimized map handling.

## Final Optimization: Arrays and Switches

Now that I understood the map overhead, I tried a more aggressive optimization. Since the nucleotide set is small and known, I replaced maps with a **fixed-size array** and a `switch` statement.

```go
func CountNucleotidesArrayBytes(dna string) map[byte]int {
  chars := [4]int{}
  for i := range len(dna) {
    switch dna[i] {
    case 'A':
      chars[0]++
    case 'C':
      chars[1]++
    case 'G':
      chars[2]++
    case 'T':
      chars[3]++
    }
  }
  return map[byte]int{
    'A': chars[0],
    'C': chars[1],
    'G': chars[2],
    'T': chars[3],
  }
}

func CountNucleotidesArrayRunes(dna string) map[rune]int {
  chars := [4]int{}
  for i := range dna {
    switch dna[i] {
    case 'A':
      chars[0]++
    case 'C':
      chars[1]++
    case 'G':
      chars[2]++
    case 'T':
      chars[3]++
    }
  }
  return map[rune]int{
    'A': chars[0],
    'C': chars[1],
    'G': chars[2],
    'T': chars[3],
  }
}
```

And the benchmarks:

```sh
$ go test -bench='^(BenchmarkCountNucleotidesArray.*|BenchmarkCountNucleotidesByByteIndexAsRune)$' -benchtime=1000000x -benchmem
goos: linux
goarch: amd64
pkg: github.com/pessolato/strmapmicrobench
cpu: 12th Gen Intel(R) Core(TM) i9-12900F
BenchmarkCountNucleotidesByByteIndexAsRune-24            1000000              1642 ns/op             192 B/op          2 allocs/op
BenchmarkCountNucleotidesArrayBytes-24                   1000000               412.9 ns/op           192 B/op          2 allocs/op
BenchmarkCountNucleotidesArrayRunes-24                   1000000               680.5 ns/op           192 B/op          2 allocs/op
PASS
ok      github.com/pessolato/strmapmicrobench   2.739s
```

The result? **Substantial performance gains**, thanks to completely bypassing map lookups.

## Conclusion

This whole detour was a great reminder: **Always benchmark and profile when performance matters**. Micro-optimizations like using bytes over runes can help, but they pale in comparison to the deeper costs of how your data structures behave at runtime.

### Key Takeaways

* **Byte iteration is slightly faster** than rune iteration.
* **Map key type matters a lot**—Go uses specialized fast paths for certain types.
* **Profiling is essential** for spotting unexpected performance bottlenecks.
* When the key space is small and fixed, **arrays + switch** can outperform maps significantly.

If you're curious, you can find all the code and benchmark results here: [https://github.com/pessolato/strmapmicrobench](https://github.com/pessolato/strmapmicrobench)

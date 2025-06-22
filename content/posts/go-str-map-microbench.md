---
title: "Microbenchmarking Go: Bytes vs Runes and the Hidden Cost of Map Keys"
date: 2025-06-22T15:03:36+02:00
draft: true
toc: false
images:
tags: ["Go", "Benchmarking", "Performance", "Microoptimization"]
---

While working through the [Go track on Exercism.org](https://exercism.org/tracks/go), I encountered an interesting performance puzzle during the **Nucleotide Count** exercise. This seemingly straightforward task led me down a rabbit hole of microbenchmarking, uncovering an insight that went far beyond string iteration: the hidden cost of map key types in Go.

## The Initial Question: Bytes or Runes?

Since the input for this exercise is a DNA sequence (a string of ASCII characters like `A`, `C`, `G`, `T`), I implemented the counting logic using a `map` to track the frequency of each nucleotide.

Almost reflexively, I began to wonder: _Should I iterate over the string using bytes (`for i := 0; i < len(s); i++`) or runes (`for _, r := range s`) for optimal performance?_  

I recalled a [comment on r/golang](https://www.reddit.com/r/golang/) that emphasized how idiomatic and efficient it is to iterate by **bytes** when dealing with ASCII-only strings. So that's what I did. The solution passed the tests, and I submitted it.

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

However, when I reviewed the “Dig Deeper” section of the exercise, I noticed the solution by `bobahop` that [iterated by runes](https://exercism.org/tracks/go/exercises/nucleotide-count/approaches/switch-statement), which outperformed mine in microbenchmarks.

This was unexpected. Intuitively, I had assumed that rune iteration, which involves decoding UTF-8 characters, should incur more overhead than raw byte iteration. I knew my solution was also slower due to checking the existence of a key instead of relying on a `switch` statement, but I also tried `bobahop`'s solution with iteration through bytes and found it performed worse than his original solution.

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

I decided to create a dedicated repository to isolate and microbenchmark the different implementations with different iteration strategies.

I started with a simple problem: iterating over a `string` input, in this case transcribing DNA to RNA.

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

As anticipated, my benchmarks showed that **byte iteration is slightly faster** than rune iteration, which confused me further.

```sh
$ go test -bench='^(BenchmarkTranscribeDnaToRnaBytes|BenchmarkTranscribeDnaToRnaRunes)$' -benchtime=1000000x -benchmem
goos: linux
goarch: amd64
pkg: github.com/pessolato/strmapmicrobench
cpu: 12th Gen Intel(R) Core(TM) i9-12900F
BenchmarkTranscribeDnaToRnaBytes-24      1000000               331.4 ns/op             0 B/op          0 allocs/op
BenchmarkTranscribeDnaToRnaRunes-24      1000000               529.3 ns/op             0 B/op          0 allocs/op
PASS
ok      github.com/pessolato/strmapmicrobench   0.864s
```

Next, I simplified the functions to count nucleotides.

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

However, I still saw significant differences in performance between the two.

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

At this point, I realized I needed more information about what was going on under the hood, so I turned to profiling.

## CPU Profiling Tells the Real Story

To resolve the discrepancy, I turned to CPU profiling. Needless to say, that’s when things started to make sense.

![Graph of CPU Profile](Profile-ByByteIndex-VS-ByRuneRange.png "Graph of CPU Profile")

As you can see, the function `CountNucleotidesByByteIndex` calls the underlying `mapassign` function, which is much slower than the `mapassign_fast32` function called by `CountNucleotidesByRuneRange`.

This made me realize I had been focused entirely on the **string iteration**, but overlooked the far more significant cost: the **map operations**. In particular, the type used for map keys has a surprising impact on performance.

### Internal Map Implementations in Go

Go’s runtime uses different internal implementations depending on the key type of a map:

- When the key is a **small value** (like a `byte`), the generic `mapassign` is invoked.
- When the key is an `int32` (which is the underlying type of a `rune`), a specialized `mapassign_fast32` function is used instead.

The latter is **significantly more optimized**. Thus, although my solution used byte iteration (which is slightly more efficient), it was coupled with a map keyed by `byte`, leading to worse overall performance than the rune-based alternative.

In short, the map key type was the dominating factor in performance, **not** the iteration method.

To experiment with that, I created two other functions: one using a `map[int16]int` to store the counts and another still iterating through bytes but converting the bytes to runes when referencing the map keys.

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

Here are the microbenchmark results:

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

The results show Go does not have a specialized function to operate on maps with keys of type `int16`, and while there is the slight overhead of converting a `byte` (`uint8`) into a `rune` (`int32`), `CountNucleotidesByByteIndexAsRune` still performed better than `CountNucleotidesByRuneRange` due to iterating through bytes in a string.

## Final Optimization: Arrays and Switches

Now conscious of the costs of map operations, I decided to try to optimize further. Since the set of nucleotides is small and well-defined, I rewrote the solution using a **fixed-size array** and a `switch` statement to index and update counts. This completely bypassed the map overhead and resulted in a much faster implementation. I kept two different implementations to compare the different iteration methods still.

```go
func CountNucleotidesArrayBytes(dna string) map[byte]int {
  chars := [4]int{0, 0, 0, 0}
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
  chars := [4]int{0, 0, 0, 0}
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

This is how they compared to the fastest implementation of counting using a map I had (`CountNucleotidesByByteIndexAsRune`):

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

## Conclusion

This experience reinforced a crucial lesson: It is important to microbenchmark and profile your functions! When performance is a priority, of course. Micro-optimizations like byte vs rune iteration do matter, but their impact can be easily dwarfed by deeper runtime behaviors—like the choice of data structure and key type.

### Key Takeaways

- **Byte iteration is faster than rune iteration**, but only marginally.
- **Map key types have a significant impact** on performance in Go, due to specialized internal methods.
- **Profiling is essential** to uncover hidden bottlenecks.
- For small, fixed-size key domains, **arrays** are often preferable to maps.

You can find the complete source code and benchmark tests in the GitHub repository:  
👉 [https://github.com/pessolato/strmapmicrobench](https://github.com/pessolato/strmapmicrobench)

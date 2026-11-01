<h1 align="center">sketch.mq</h1>

> [!WARNING]
> This project is currently under active development. APIs and features may change without notice.

> [!NOTE]
> This module needs the development version of mq, which is not released yet: it uses the `hash` and `bit_length` builtins.

Probabilistic summaries of a stream of items for [mq](https://github.com/harehare/mq), as a module. They keep a fixed amount of memory however many items go through, and answer with a small, known error.

Use them when you need fixed-size summaries that can be merged, for example one sketch per file combined into a total. To count items within a single run, a dict is simpler and exact.

| Module | Answers |
| --- | --- |
| `sketch::hll` | how many distinct items have been seen (HyperLogLog) |
| `sketch::cms` | how many times each item has been seen (Count-Min Sketch) |

## Installation

Copy `sketch.mq` to your mq module directory, or place it anywhere and reference it with `-L`.

```sh
cp sketch.mq ~/.local/mq/config/
```

HTTP imports are disabled by default; pass `--allow-http-import` to import directly from GitHub:

```sh
mq --allow-http-import -I null 'import "github.com/harehare/sketch.mq" | sketch::hll::count(sketch::hll::new(10))'
```

## Usage

### Distinct counts

```sh
mq -L . -I null '
  import "sketch"
  | let h = sketch::hll::add_all(sketch::hll::new(10), map(range(0, 999), fn(i): "user-" + to_string(i % 700);))
  | sketch::hll::count(h)
'
# 702
```

1000 items covering 700 distinct values give an estimate of 702.

| Function | Description |
| --- | --- |
| `hll::new(precision)` | empty counter with `2^precision` registers, for a precision from 4 to 16 |
| `hll::add(h, item)` / `hll::add_all(h, items)` | add one item or an array of items |
| `hll::count(h)` | estimated number of distinct items |
| `hll::merge(a, b)` | counter for the items of both, of the same precision |
| `hll::error_rate(h)` | the standard error to expect, about `1.04 / sqrt(registers)` |
| `hll::precision(h)` | the precision |

A precision of 10 uses 1024 registers and is off by about 3%; 14 uses 16384 and is off by about 0.8%. Small counts are close to exact.

### Frequencies

```sh
mq -L . -I null '
  import "sketch"
  | let s = sketch::cms::add_all(sketch::cms::for_error(0.01, 0.01), ["a", "b", "a", "c", "a"])
  | [sketch::cms::estimate(s, "a"), sketch::cms::estimate(s, "never"), sketch::cms::total(s)]
'
# [3, 0, 5]
```

| Function | Description |
| --- | --- |
| `cms::new(width, depth)` | empty sketch of `depth` rows of `width` counters |
| `cms::for_error(epsilon, delta)` | sized so an estimate exceeds the true count by at most `epsilon` times the total, with probability `1 - delta` |
| `cms::add(s, item)` / `cms::add_many(s, item, amount)` / `cms::add_all(s, items)` | count occurrences |
| `cms::estimate(s, item)` | estimated count, never below the true count |
| `cms::total(s)` | number of occurrences added |
| `cms::merge(a, b)` | sketch for the counts of both, of the same shape |

## Notes

- Items are compared by the string `to_string` gives for them, so `42` and `"42"` are the same item.
- A sketch is a plain array and every operation returns the updated sketch. Write `h = sketch::hll::add(h, x)` and do not read the old `h` afterwards, so the update happens in place.
- `hll::add`, `cms::add` and `cms::estimate` take constant time. `hll::count` and the merges take time proportional to the size of the sketch.
- Items are hashed with the `hash` builtin (SipHash-2-4), so results are the same on every run and platform.

## Development

```sh
mq-test sketch_tests.mq
mq-bench -L . sketch_bench.mq
```

One of the goals of this module is also to check mq for performance problems, so it comes with a benchmark in `sketch_bench.mq`.

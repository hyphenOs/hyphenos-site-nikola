.. title: Rust Performance Measurement Initial Steps
.. slug: rust-performance-measurement-initial-steps
.. date: 2021-08-09 16:43:00 UTC+05:30
.. tags: Rust, Performance, FlameGraphs
.. author: Abhijit Gadgil
.. link:
.. description:
.. type:
.. status:
.. summary: This blog post explores the Rust ecosystem tools for performance measurement and explains how the tools were used in solving a performance issue in a crate that we are developing.

# Introduction

While working on [`wishpy`](https://pypi.org/project/wishpy/){:target="\_blank"} one of the things that I realized was, [`Wireshark`](https://www.wireshark.org/){:target="\_blank"} is primarily meant to be used as a tool. While, Wireshark provides programmatic API (C based), they are not very convenient to use. One of the Rust project ideas that I had was - to develop a packet dissection framework, that is API friendly. There is a Go based [`gopacket`](https://github.com/google/gopacket/){:target="\_blank"} which comes quite close to what I had in mind. I had a rough idea of how to get it working in Rust and I started upon implementing it in a crate called [`scalpel`](https://github.com/gabhijit/scalpel){:target="\_blank"}. It's still a work in progress and this blog post is not about it _per se_. Since this particular code is going to be sitting at a very low level in a Packet Analysis stack, the actual dissection should be really fast. My goal to start with was to be at-least at par, if not better with `gopacket`'s performance without any optimizations. For a couple of basic dissectors like `IPv4` and `IPv6`, it was doing actually substantially better - nearly twice as fast compared to `gopacket`, and this looked quite promising, but for one particular benchmark for a DNS Packet, the performance more than twice slower than the `gopacket`'s. The actual reason for this ended up being something else and not just the dissection performance, but identifying the root cause helped in better understanding at-least some parts of Rust's ecosystem for running benchmarks and instrumentation of running code. Next, we'll look at some of those tools and also how these tools helped identify and reason about the performance issues and what steps were taken to solve the issue.

# Background

Before we get into the details of the actual changes and the eventual improvements, a quick look at the ecosystem. There is a cargo command called `cargo bench` that can be used to run benchmark programs. [`criterion`](https://bheisler.github.io/criterion.rs/book/index.html){:target="\_blank"} is a micro-benchmarking tool that can be used with `cargo bench`. The documentation of `criterion` is quite good and we will not look into the details of how that is to be used. We need to add it as one of the `dev-dependencies` and add modules inside the `benches` directory of your crate and then `cargo bench` will take over.

While `cargo bench` is useful for running benchmarks, to be able to look underneath, we need additional instrumentation mechanism. Brendan Greg's [Flamegraphs](https://www.brendangregg.com/flamegraphs.html){:target="\_blank"} is an excellent tool for this. It can be used to generate a nice SVG which shows the frequently used code paths along with their call stacks. This can show which parts of the program are spending more time. There is a [Rust port of FlameGraph](https://github.com/jonhoo/inferno){:target="\_blank"}, which is integrated with cargo toolchain and one can run a command called `cargo flamegraph`, that generates the flame graph for running a particular benchmark.

So the plan was to generate FlameGraph for the benchmark that was performing poorly and identifying the issues and see how they can be addressed if at all. Also, the idea was not to actually perform any 'hand optimizations', but to simply avoid parts that were poorly implemented (like too many allocations for example) and see if they can be improved and how much.

# A Note about DNS Records

The code that we are exploring is for the dissection of DNS packets, before we actually look into the code, let's quickly understand a bit about how records are compressed in the DNS packets as this particular part of the code was problematic.

In a DNS packet, DNS records for the hostnames are compressed by not repeating the parts of the hostname again if multiple records are present and simply providing the pointer to the actual records, for instance let's say there are four name server records for `ns1.example.com`, `ns2.example.com`, `ns3.example.com` and `ns4.example.com`, this is encoded in a packet as `ns1.example.com` and for `ns2`, `ns3` and `ns4` only the pointer to the `.example.com` is kept. [This section](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.4){:target="\_blank"} in the DNS RFC has more details about it.


# First Implementation and Issues

We are going to look at a specific function that is obtaining the DNS names from the records (potentially compressed) in a DNS Packet. This function is `dns_name_from_u8`. The function looks like -

```rust
 fn dns_name_from_u8(
        bytes: &[u8],
        start: usize,
        remaining: usize,
    ) -> Result<(DNSName, usize), Error> {
        fn labels_from_offset(
            bytes: &[u8],
            offset: usize,
            mut remaining: usize,
            check_remaining: bool,
        ) -> Result<(Vec<Vec<u8>>, usize), Error> {
           ...
           ...
        }

        let (labels, consumed) = labels_from_offset(bytes, start, remaining, true)?;

        Ok((DNSName(labels.into_iter().flatten().collect()), consumed))
    }
```
The actual code for this [implementation can be found here](https://github.com/gabhijit/scalpel/blob/07fde032f10b617eae9feaddc2eb0a61ea54b530/src/layers/dns.rs#L90){:target="\_blank"}.

So what the code was essentially doing was recursively calling the `labels_from_offset`, where each call to `labels_from_offset` was returning a `Vec<u8>` and finally all the vectors were flattened into a `Vec<u8>`.


This part looked like the culprit and indeed the FlameGraph did indicate that a chunk of time was being spent here. See below -


![FlameGraph 1 - FlameGraph for `Vec<Vec<u8>>`](/blog/images/flamegraph_1.png "FlameGraph 1")

## Benchmark Results

The `cargo bench` results for this particular code were -

| Benchmark | Mean Time per Iteration |
|:--|:--:|
| Parse_DNS_AAAA | 2.7950 us {{% emoji heavy_exclamation_mark_symbol %}} |
| Parse_DNS | 525.86 ns |
| Parse_DNS_Regression_Packet | 575.52 ns |

Note: In the table above only the mean value (central value in three values that `cargo bench` reports) is shown. There are many other details that `cargo bench` provides, but are omitted here since we are mainly looking at one particular parameter, that is the run-time of the benchmark. Also, these results should be compared for relative performance as actual numbers will vary from machine to machine.

# Iteration 1 - A False Start

To improve the performance, the recursive call to the `labels_from_offset` path was avoided, this looked like improved the performance, but this actually resulted in a breaking test case and hence this approach was discarded. In hindsight, it was 'obvious' that this made little sense because the nature of the 'compression' is indeed recursive, so recursive calls cannot be avoided.

# Iteration 2 - Flattening the Vector

This `Vec<Vec<u8>>` can be avoided by using a 'flattened' vector and thus avoiding the flattening cost. This iteration was tried next. This helped improve the performance quite a bit, nearly twice for the worst performing case and by about 20-30 percent in other cases. This was understandable because in the other cases there were not as many compressions and hence that part of the code was not getting hit often. Also, instead of allocating a `Vec` with no capacity and growing it, some `capacity` was reserved, which helped as well.

The actual code for this [implementation can be found here](https://github.com/gabhijit/scalpel/blob/891355f8ec7de38c778e11fbdfdfb328927a2237/src/layers/dns.rs#L90){:target="\_blank"}.

## Benchmark Results

Summary results after running `cargo bench` after the above changes. As can be seen the problematic case performance improved substantially.

| Benchmark | Mean Time per Iteration |
|:--|:--:|
| Parse_DNS_AAAA  | 1.2503 us {{% emoji white_heavy_checkmark %}} |
| Parse_DNS       | 394.31 ns |
| Parse_DNS_Regression_Packet | 462.30 ns |

<br/>
Also, as can be seen in the flamegraph below - Most of the time spent in Vector Alloc is gone now.

![FlameGraph 2 - FlameGraph for `Vec<u8>`](/blog/images/flamegraph_2.png "FlameGraph 2")

# Iteration 3 - Avoiding the Unused Vector Allocations

One more observation was, in every recursive call, we were allocating a vector and it could go unused if that recursive call didn't end up in extending the vector. This could as well be avoided by allocating and reserving the vector in the main calling function (`dns_name_from_u8`) and then passing a mutable reference to it when calling `labels_from_offset`. Thus for every record there is only one allocation. This improved the performance of the problematic case by another 20-30%, while the performance for the other two cases was more or less unchanged (understandably, because this performance degradation was a function of number of calls to `dns_name_from_u8`).

The actual code for this [implementation can be found here](https://github.com/gabhijit/scalpel/blob/master/src/layers/dns.rs#L131){:target="\_blank"}.

## Benchmark Results

| Benchmark | Mean Time per Iteration |
|:--:|:--:|
| Parse_DNS_AAAA | 919.64 ns |
| Parse_DNS	     | 384.97 ns |
| Parse_DNS_Regression_Packet | 450.01 ns |

<br/>
As can be seen from the FlameGraph, there is no visible difference in the FlameGraph between this case and the previous case.

![FlameGraph 3 - FlameGraph for `Vec<u8>`](/blog/images/flamegraph_3.png "FlameGraph 3")

# Key Takeaways

Key takeaways from these iterations can be summarized as follows -

1. Avoid flattening the vectors in a fast path as far as possible. Note: This may not be always possible, but if there is a way to avoid this, it's perhaps a good idea.
2. Also, avoid all unnecessary allocations and consider reserving a capacity if it makes sense. What 'reserved' capacity is a sweet spot can be determined by experimenting with different values.
3. Tools like FlameGraph provide excellent visible feedback about our reasoning about performance and help us identify areas where bulk of the time in the code is spent if any and one can then focus on improving those areas of the code.

# Postscript

Even after these 'optimizations' (they should rather be called - improvements than optimizations), the code finally was not doing just as well as the `gopacket` code, which still didn't make a whole lot of sense as most other benchmarks were running substantially better. It turned out, that the `gopacket` benchmark was reusing an allocated slice and hence appeared to run faster. When the same benchmark in `gopacket` was changed to run like the benchmark above, the results were consistent, the benchmark running substantially faster. The below table shows the comparison for benchmarks between `gopacket` implementation and `scalpel` implementation. While it might look like `scalpel` is doing twice better than `gopacket` for these benchmarks, I would not jump to the conclusion and shout this is twice as fast, but surely it's promising enough (in fact more than promising enough) to be pursued further.


| Benchmark | `gopacket`\*| `gopacket`+ | `scalpel` |
|:--|:--:|:--:|:--:|
| Parse_DNS_AAAA | 4566.2 ns | 644.3 ns | 919.64 ns {{% emoji white_heavy_checkmark %}}|
| Parse_DNS | 867.8 ns | N/A | 384.97 ns {{% emoji white_heavy_checkmark %}}|
| Parse_DNS_Regression_Packet | 1094 ns | N/A | 450.01 ns {{% emoji white_heavy_checkmark %}}|

\* - `gopacket` benchmark modified for comparison with `scalpel`.
<br/>
\+ - The actual `gopacket` benchmark (reusing the allocated slice).

# Scalpel Looking for Contributions

Scalpel is looking for help from people to contribute to write dissectors for more protocols. Please check the scalpel project on Github.

1. **`scalpel`** - [Packet Dissection and Sculpting in Rust](https://github.com/gabhijit/scalpel){:target="\_blank"}

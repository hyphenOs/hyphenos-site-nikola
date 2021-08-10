.. title: Rust Performance Measurement Initial Steps
.. slug: rust-performance-measurement-initial-steps
.. date: 2021-08-09 16:43:00 UTC+05:30
.. tags: Rust, Performance, FlameGraphs
.. author: Abhijit Gadgil
.. link:
.. description:
.. type:
.. status:
.. summary: This blog post explores the Rust ecosystem tools for performance measurement and explains how the tools were used in identifying a performance issue in a crate that we are developing.

# Introduction

While working on [`wishpy`](https://pypi.org/project/wishpy/) one of the things that I realized was, [`Wireshark`](https://www.wireshark.org/) is primarily meant to be used as a tool. While, Wireshark provides programmatic API (C based), they are not very convenient to use. I wanted to develop a packet dissection framework, that is API friendly. [`gopacket`](https://github.com/google/gopacket/) comes quite closer to what I was thinking. I had a rough idea of how to get it working in Rust and I started upon implementing it in a crate called [`scalpel`](https://github.com/gabhijit/scalpel). It's still a work in progress and this blog post is not about it per se. Since this particular code is going to be sitting at a very low level in a Packet Analysis stack, the actual dissection should be really fast. My goal to start with was to be at-least at par, if not better with `gopacket`'s performance without any optimizations. For a couple of basic dissectors like `IPv4` and `IPv6`, it was doing actually substantially better - nearly twice as fast compared to `gopacket`, and this looked quite promising, but for one particular benchmark for a DNS Packet, the performance more than twice slower than the `gopacket`'s. The actual reason for this ended up being something else and not just the dissection performance, but identifying the root cause helped in better understanding at-least some parts of Rust's ecosystem for running benchmarks and some instrumentation of running code. Next, we'll look at some of those tools and also how these tools helped identify and reason about the performance issues and what steps were taken to solve the issue.

# Background

Before we get into the details of the actual changes and the eventual improvements, a quick look at the ecosystem. There is a cargo command called `cargo bench` that can be used to run benchmark programs. [`criterion`](https://bheisler.github.io/criterion.rs/book/index.html) is a micro-benchmarking tool that can be used with `cargo bench`. The documentation of `criterion` is quite good and we will not look into the details of how that is to be used. We need to add it as one of the `dev-dependencies` and add modules inside the `benches` directory of your crate and then `cargo bench` will take over.

While `cargo bench` is useful for running benchmarks, to be able to look underneath, we need additional instrumentation mechanism. Brendan Greg's [Flamegraphs](https://www.brendangregg.com/flamegraphs.html) is an excellent tool for this. It can be used to generate a nice SVG which shows the frequently used code paths in the code along with their call stacks. This can show which parts of the program are spending more time. There is a [Rust port of FlameGraph](https://github.com/jonhoo/inferno), which is integrated with cargo toolchain and one can run a command called `cargo flamegraph`, that generates the flame graph for running a particular benchmark.

So the plan was to generate FlameGraph for the benchmark that was performing poorly and identifying the issues and see how they can be addressed if at all. Also, the idea was not to actually perform any 'hand optimizations', but to simply avoid parts that were poorly implemented (like too many allocations for example) and see if they can be improved and how much.

# A Note about DNS Records

In a DNS packet, DNS records for the hostnames are compressed by not repeating the parts of the hostname again if multiple records are present and simply providing the pointer to the actual records, for instance let's say there are four name server records for `ns1.example.com`, `ns2.example.com`, `ns3.example.com` and `ns4.example.com`, this is encoded in a packet as `ns1.example.com` and for `ns2`, `ns3` and `ns4` only the pointer to the `.example.com` is kept. [This section](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.4) in the DNS RFC has more details about it.

One observation about the benchmark was whenever the compression was high, that is more number of records in a DNS message having common prefix, the performance was very bad. This also identified that the record handling to be a possible problem area.

# First Implementation and Issues

While dissecting a DNS packet, the DNS records were dissected using a function that looked like -

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
So what the code was essentially doing was recursively calling the `labels_from_offset`, where each call to `labels_from_offset` was returning a `Vec<u8>` and finally all the vectors were flattened into a `Vec<u8>`.

This part looked like the culprit and indeed the FlameGraph did indicate that a chunk of time was being spent here. See below -

```shell

Benchmark Results

Parse_DNS               time:   [525.33 ns 525.86 ns 526.42 ns]
                        change: [+1.9676% +2.2988% +2.6414%] (p = 0.00 < 0.05)
                        Performance has regressed.
Found 5 outliers among 100 measurements (5.00%)
  1 (1.00%) high mild
  4 (4.00%) high severe

Parse_DNS_AAAA          time:   [2.7863 us 2.7950 us 2.8053 us]
                        change: [-81.293% -31.593% +220.45%] (p = 0.78 > 0.05)
                        No change in performance detected.
Found 6 outliers among 100 measurements (6.00%)
  2 (2.00%) high mild
  4 (4.00%) high severe

Parse_DNS_Regression_Packet
                        time:   [573.87 ns 575.52 ns 577.39 ns]
                        change: [+2.2685% +2.6636% +3.0605%] (p = 0.00 < 0.05)
                        Performance has regressed.
Found 4 outliers among 100 measurements (4.00%)
  3 (3.00%) high mild
  1 (1.00%) high severe

```



# Iteration 1 - A False Start

To improve the performance, the recursive call to the `labels_from_offset` path was avoided, but this actually resulted in a breaking test case and that path was not taken forward. In hindsight, it was 'obvious' that this made little sense because the nature of the 'compression' is indeed recursive, so recursive calls cannot be avoided.

# Iteration 2 - Flattening the Vector

This kind of `Vec<Vec<u8>>` can be avoided by using a 'flattened' vector and thus avoiding the flattening cost. This iteration was tried next. This helped improve the performance quite a bit, nearly twice for the worst performing case and by about 20-30 percent in other cases. This was understandable because in the other cases there were not as many compressions and hence that part of the code was not getting hit often. Also, instead of allocating a `Vec` with no capacity and growing it, some `capacity` was reserved, which helped as well. The exact contribution cannot attributed in the benchmark results below though.

```shell


Parse_DNS_AAAA          time:   [1.2435 us 1.2503 us 1.2582 us]
                        change: [-84.277% +35.931% +474.47%] (p = 0.82 > 0.05)
                        No change in performance detected.
Found 7 outliers among 100 measurements (7.00%)
  7 (7.00%) high severe

Parse_DNS               time:   [393.85 ns 394.31 ns 394.82 ns]
                        change: [-25.316% -25.046% -24.798%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 10 outliers among 100 measurements (10.00%)
  1 (1.00%) low mild
  4 (4.00%) high mild
  5 (5.00%) high severe

Parse_DNS_Regression_Packet
                        time:   [461.56 ns 462.30 ns 463.05 ns]
                        change: [-20.156% -19.833% -19.541%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 4 outliers among 100 measurements (4.00%)
  1 (1.00%) low mild
  2 (2.00%) high mild
  1 (1.00%) high severe

```
Also, as can be seen in the flamegraph below - Most of the time spent in Vector Alloc is gone now.

Iteration 2 - Avoiding the Unused Vector Allocations

One more observation was, in every recursive call, we were allocating a vector and it could go unused if that recursive call didn't end up in extending the vector. This could as well be avoided by allocating and reserving the vector in the main calling function (`dns_name_from_u8`) and then passing a mutable reference to it when calling `labels_from_offset`. Thus for every record there is only one allocation. This improved the performance of the problematic case by another 20-30%, while the performance for the other two cases was more or less unchanged (understandably, because this performance degradation was a function of number of calls to `dns_name_from_u8`).

```shell


Parse_DNS_AAAA          time:   [910.67 ns 919.64 ns 928.11 ns]
                        change: [-94.271% -0.5954% +1201.3%] (p = 0.93 > 0.05)
                        No change in performance detected.
Found 6 outliers among 100 measurements (6.00%)
  2 (2.00%) high mild
  4 (4.00%) high severe

Parse_DNS               time:   [384.53 ns 384.97 ns 385.41 ns]
                        change: [-2.7541% -2.4101% -2.0651%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 5 outliers among 100 measurements (5.00%)
  1 (1.00%) low mild
  2 (2.00%) high mild
  2 (2.00%) high severe

Parse_DNS_Regression_Packet
                        time:   [449.51 ns 450.01 ns 450.53 ns]
                        change: [-2.7310% -2.4289% -2.1226%] (p = 0.00 < 0.05)
                        Performance has improved.
Found 10 outliers among 100 measurements (10.00%)
  2 (2.00%) low mild
  5 (5.00%) high mild
  3 (3.00%) high severe

```
As can be seen from the FlameGraph, there is no visible difference in the FlameGraph between this case and the previous case.


# Key Takeaways

Key takeaways from these iterations can be summarized as follows -

1. Avoid flattening the vectors in a fast path as far as possible. Note: This may not be always possible, but if there is a way to avoid this, it's perhaps a good idea.
2. Also, avoid all unnecessary allocations and consider reserving a capacity if it makes sense. What 'reserved' capacity is a sweet spot can be determined by experimenting with different values.
3. Tools like FlameGraph provide excellent visible feedback about our reasoning about performance and help us identify areas where bulk of the time in the code is spent if any and one can then focus on improving those areas of the code.

# Postscript

Even after these 'optimizations' (they should rather be called - improvements than optimizations), the code finally was doing just as well as the `gopacket` code, which still didn't make a whole lot of sense as most other benchmarks were running substantially better. It turned out, that the `gopacket` benchmark was reusing an allocated slice and hence appeared to run faster. When the same benchmark in `gopacket` was changed to run like the benchmark above, the results were consistent, the benchmark running substantially faster, sometimes almost twice as fast as `gopacket`.




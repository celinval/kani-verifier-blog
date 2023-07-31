---
layout: post
title:  "Optimizing Kani"
---

Kani is a verification tool that can help you systematically test properties about your Rust code.
To learn more about Kani, check out the [Kani tutorial](https://model-checking.github.io/kani/kani-tutorial.html) and our [previous blog posts](https://model-checking.github.io/kani-verifier-blog/).

Over the past 9 months we have optimised Kani at different levels to either improve general performance or solve some specific performance problems encountered on some user benchmarks.

The performance improvements are illustrated here on s2n-quic reference (Synthesis of gains achieved between Nov 2022 and July 2023)

@karkhaz can we place an analysis of the benchcomp results in this section ?

- `kani --solver=minisat` vs `kani --solver=cadical` (using the kani 0.32 and cbmc 5.88.0)
- `kani --write-json-symtab` vs `kani` (using the kani 0.32 and cbmc 5.88.0), isolating the the codegen + goto model export time.
- `kani` `v0.32` running with `cbmc  v5.70.0` vs cbmc `cbmc v5.88.0` (for union field sensitivity).

In this post, we'll discuss three optimizations enabled through different components of Kani. But before that, we'll provide a high-level overview of Kani's architecture.

# Kani Architecture Overview

First, let's briefly introduce you to Kani's high level architecture to understand where modifications were performed.

From the user perspective, independently on how you invoke Kani, via cargo or directly invoking the Kani binary, the verification process is very similar. The user invokes Kani and provides cargo packages or a Rust crate that should be verified. Kani will execute until it can check all existing harnesses, and it will report whether each harnesses was successfully verified or failed.

Internally, this verification process is a bit more complicated, and can be split into three main stages:

1. **Compilation:** The Rust crate under verification and its dependencies are compiled into a program in a format that's more suitable to verification.
2. **Symbolic execution:** This program is then symbolically executed in order to generate a single logic formula that represents all possible execution paths and the properties to be checked.
3. **Satisfiability solving:** This logic formula is then solved by employing a SAT or SMT solver that either finds a combination of concrete values that can satisfy the formula (a counter example to a safety property or user assertion exists), or prove that no assignement can satisfy the formula (the program is safe and all assertion hold).

Kani in fact employs a collection of tools to perform the different stages of the verification. Kani's main process is called `kani-driver`, and its main purpose is to orchestrate the execution and communication of these other tools:

1. The compilation statge is done mostly* by the kani-compiler, which is an extension of the Rust compiler that we have developed. The kani-compiler will generate a `goto-program` by combining all the logic that is reachable from a harness.
2. For the symbolic execution stage Kani invokes [CBMC](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwiet5XSyqiAAxUOEFkFHeUiB6IQFnoECBgQAQ&url=https%3A%2F%2Fwww.cprover.org%2Fcbmc%2F&usg=AOvVaw3LFf7eUtKK6bQEkDWKE29S&opi=89978449).
3. The satisfiability checking stage is performed by CBMC itself, by invoking a [satisfiability (SAT) solver](https://en.wikipedia.org/wiki/SAT_solver) such as [MiniSat](http://minisat.se/).

<!-- double-wrapping means we can click to enlarge -->
<a href="{{site.baseurl | prepend: site.url}}/assets/images/kani-high-level.png"><img src="{{arch itecture.baseurl | prepend: site.url}}/assets/images/kani-high-level.png" alt="Kani architecture" /></a>

To verify [Cargo](https://doc.rust-lang.org/stable/cargo/) packages, Kani employs Cargo to correctly build all the dependencies before translating them to goto-programs.

The verification problem is computationally hard, so any optimisation that can brings benefits overall, or help in some specific situation is worth having.

# Supporting Multiple SAT Solvers

SAT solving is typically the most time-consuming part of a Kani run. There is a large number of SAT solvers available, whose performance can vary widely depending on the specific type of formula, and it can be very helpful to be able to try different SAT solvers on the same problem to determine which one performs best.

By default, CBMC uses the [MiniSat](http://minisat.se/) SAT solver.
While CBMC can be configured to use a different SAT solver at build time, having to rebuild it to switch SAT solvers is inconvenient.
Thus, we introduced an enhancement to CBMC's build system that allows CBMC to be built with multiple SAT solvers, so that the user can select one of them at _runtime_ via an option (`--sat-solver`)[^1].

With this enhancement in CBMC, and to ease the selection of the SAT solver for Kani users, we've introduced a Kani attribute, `kani::solver`, that can be used to specify the SAT solver to use for each harness.
We've also introduced a global command-line switch, `--solver <SOLVER>`, that overrides the harness `kani::solver` attribute.

For instance, one can configure Kani to use [CaDiCaL](https://github.com/arminbiere/cadical) as follows:

```rust
#[kani::proof]
#[kani::solver(cadical)] // <--- Use CaDiCaL for this harness
fn my_harness() { ... }
```

Changing the solver can result in orders of magnitude performance difference. Thus, we encourage users to try different solvers to find the one that performs the best on their harness. At the time of writing, the following three solvers are supported out of the box by Kani: `minisat` (the default solver), `cadical`, and `kissat` ([Kissat](https://github.com/arminbiere/kissat)). Kani also allows using other SAT solvers available as standalone binaries in your system `PATH`. This can be done using:

```rust
#[kani::solver(bin="<SAT_SOLVER_BINARY>")]                                                                                 `
```

An example of a SAT solver that we've found effective for certain classes of programs (e.g. ones involving cryptographic operations) is [CryptoMiniSat](https://github.com/msoos/cryptominisat). After installing CryptoMiniSat and adding the binary to your path, you can configure Kani to use it via:

```rust
#[kani::solver(bin="cryptominisat5")]
```

The table below shows the runtimes in seconds obtained using the global `--solver` switch with Minisat, CaDiCaL and Kissat on the `s2n-quic-core` crate, with a timeout of 30 minutes. The `Speedup` columns give the speedup factor with respect to Minisat, and the `Fastest` column gives the fastest solver for each benchmark.

TODO add kani and cbmc versions.

```
Harness                                                    Minisat  CaDiCaL  Kissat  CaDiCaL  Kissat   Fastest
                                                                                     Speedup  Speedup
recovery::rtt_estimator::test::weighted_average_test...    TIMEOUT  68.99    40.13   N/A      N/A      Kissat
sync::cursor::tests::oracle_test...                        TIMEOUT  255.96   217.09  N/A      N/A      Kissat
random::tests::gen_range_biased_test...                    1459.82  6.80     5.46    214.78   267.49   Kissat
sync::spsc::tests::alloc_test...                           1003.92  70.12    62.75   14.32    16.00    Kissat
slice::tests::vectored_copy_fuzz_test...                   942.01   69.18    103.96  13.62    9.06     CaDiCaL
message::cmsg::tests::round_trip_test...                   933.72   357.60   324.76  2.61     2.88     CaDiCaL
task::completion_to_tx::assign::tests::assignment_test...  279.75   25.51    54.58   10.97    5.13     CaDiCaL
inet::ipv6::tests::header_getter_setter_test...            34.68    35.08    34.73   0.99     1.00     Minisat
varint::tests::eight_byte_sequence_test...                 28.35    9.87     9.83    2.87     2.88     Kissat
varint::tests::four_byte_sequence_test...                  15.08    9.59     9.37    1.57     1.61     Kissat
packet::number::tests::round_trip...                       13.46    13.40    14.71   1.00     0.92     CaDiCaL
inet::ipv4::tests::header_getter_setter_test...            13.06    13.57    13.20   0.96     0.99     Minisat
varint::tests::round_trip_values_test...                   12.59    12.54    14.23   1.00     0.88     CaDiCaL
frame::stream::tests::try_fit_test...                      12.08    22.34    18.18   0.54     0.66     Minisat
varint::tests::two_byte_sequence_test...                   11.57    8.61     8.50    1.34     1.36     Kissat
inet::ipv6::tests::scope_test...                           11.34    11.60    12.78   0.98     0.89     Minisat
inet::checksum::tests::differential...                     10.55    16.21    14.28   0.65     0.74     Minisat
varint::tests::one_byte_sequence_test...                   9.32     7.41     7.53    1.26     1.24     CaDiCaL
xdp::decoder::tests::decode_test...                        8.08     4.85     5.79    1.67     1.40     CaDiCaL
message::msg::tests::address_inverse_pair_test...          5.89     6.64     7.55    0.89     0.78     Minisat
message::cmsg::tests::iter_test...                         4.94     12.46    10.25   0.40     0.48     Minisat
frame::crypto::tests::try_fit_test...                      3.42     5.17     4.72    0.66     0.72     Minisat
inet::ipv4::tests::scope_test...                           2.76     2.82     3.50    0.98     0.79     Minisat
interval_set::tests::interval_set_inset_range_test...      2.31     2.41     2.31    0.96     1.00     Minisat
packet::number::tests::rfc_differential_test...            2.02     2.49     2.44    0.81     0.83     Minisat
packet::number::map::tests::insert_value...                1.70     1.72     1.75    0.99     0.97     Minisat
packet::number::tests::truncate_expand_test...             1.38     1.62     1.61    0.85     0.86     Minisat
varint::tests::checked_ops_test...                         1.00     1.77     1.21    0.56     0.82     Minisat
packet::number::sliding_window::test::insert_test...       0.88     0.94     1.03    0.94     0.86     Minisat
varint::tests::table_differential_test...                  0.61     0.68     0.63    0.90     0.98     Minisat
ct::tests::mul...                                          0.57     0.47     0.54    1.21     1.05     CaDiCaL
stream::iter::fuzz_target::fuzz_builder...                 0.52     0.53     0.55    0.97     0.94     Minisat
ct::tests::sub...                                          0.49     0.42     0.46    1.15     1.07     CaDiCaL
ct::tests::add...                                          0.45     0.42     0.45    1.06     0.99     CaDiCaL
ct::tests::rem...                                          0.43     0.71     0.49    0.60     0.87     Minisat
ct::tests::div...                                          0.42     0.58     0.66    0.73     0.65     Minisat
ct::tests::ct_lt...                                        0.42     0.41     0.42    1.02     1.00     CaDiCaL
ct::tests::ct_le...                                        0.41     0.40     0.42    1.02     0.98     CaDiCaL
ct::tests::ct_gt...                                        0.40     0.40     0.41    1.00     0.98     CaDiCaL
ct::tests::ct_ge...                                        0.40     0.40     0.42    1.01     0.96     CaDiCaL
packet::number::tests::example_test...                     0.22     0.22     0.22    1.00     0.96     CaDiCaL
```

We see that Kissat and CaDiCaL can solve benchmarks that would timeout with Minisat, and provide significant speedups on some other benchmarks. We also see that Minisat remains the fastest solver for harnesses that already run in a matter of seconds.

By picking the best solver for each harness using the `kani::solver` attribute, we can can bring the total cumulative runtime from 8431s to 924s (counting timeouts as 1800s), while solving two more harnesses. Great savings if your harnesses are run in CI!

Maybe drop that reference ?
This [this pull-request](https://github.com/aws/s2n-quic/pull/1771) on the [s2n-quic repository](https://github.com/aws/s2n-quic) shows that using `cadical` instead of the default `minisat` brought speedups in the range of x1.5-x28 on a number of benchmarks.

# Adding Direct Export of GOTO Binaries

The Kani compiler translates a Rust program into a `goto-program` stored in a  *symbol table*  that lists all symbols manipulated by the program together with their source locations and definitions (types symbols and their definitions, functions symbols and their bodies, static symbols and their initial value, etc.). The symbol table data consists mainly of abstract syntax trees. Kani originally serialized symbol tables to JSON and used the CBMC utility `symtab2gb`  to translate the JSON symbol table to GOTO binary format, which is the format that CBMC actually expects as input. The JSON to GOTO binary conversion was one of the most time and memory consuming steps in the compilation stage. We implemented direct GOTO binary serialisation in Kani (see code [here](https://github.com/model-checking/kani/blob/main/cprover_bindings/src/irep/goto_binary_serde.rs)), which allows us to skip the costly invocation of `symtab2gb`. Kani can now perform the codegen and symbol table export 2-4x faster than before, sometimes with a reduction in peak memory consumption depending on the crate.



The table below reports time and memory consumption required for  codegen and symbol table export steps for three crates of the `s2n-quic` project, run using `kani --tests --only-codegen`.
The time includes Rust to MIR compilation time, GOTO code generation from MIR time and GOTO binary export, either through JSON and `symtab2gb`, or directly to GOTO binary.

| Crate               | JSON export    |               |                                | GOTO binary export |               |                                |
| ------------------- | -------------- | ------------- | ------------------------------ | ------------------ | ------------- | ------------------------------ |
|                     | System time(s) | User time (s) | Maximum resident set size (kb) | System time(s)     | User time (s) | Maximum resident set size (kb) |
| `s2n-quic-core`     | 29             | 198           | 669868                         | 22                 | 111           | 815076                         |
| `s2n-quic-platform` | 13             | 99            | 449652                         | 13                 | 91            | 450128                         |
| `s2n-quic-xdp`      | 11             | 92            | 450128                         | 11                 | 95            | 450708                         |


We can see that for `s2n-quic-core`, which contains over 35 individual harnesses, the User time required to compile and export is reduced by almost 45% 199s to 111s, for a 20% memory consumption increase. For the two other crates which contain respectively 3 and 1 harnesses, the User time gains are lower at 10% and even negative at -4%, for a negligible memory consumption increase.


Looking in more detail at individual harnesses (only for GOTO code generation and GOTO binary export time, without Rust to MIR compilation) we see that the time for JSON symbol table export used to dominate the GOTO code generation, and that direct GOTO binary export is now faster than GOTO code generation from MIR.

| Harness                                                     | GOTO codegen time (s) | JSON export (s) | GOTO-binary export (s) | export speedup | codegen + export speedup |
| ----------------------------------------------------------- | --------------------- | --------------- | ---------------------- | -------------- | ------------------------ |
| `ct::tests::ct_ge...`                                       | 0,32                  | 1,48            | 0,17                   | 8,70           | 3,67                     |
| `ct::tests::ct_gt...`                                       | 0,26                  | 1,32            | 0,14                   | 9,42           | 3,95                     |
| `ct::tests::ct_le...`                                       | 0,26                  | 1,32            | 0,14                   | 9,42           | 3,95                     |
| `ct::tests::ct_lt...`                                       | 0,26                  | 1,33            | 0,14                   | 9,50           | 3,97                     |
| `ct::tests::rem...`                                         | 0,26                  | 1,32            | 0,16                   | 8,25           | 3,76                     |
| `ct::tests::sub...`                                         | 0,26                  | 1,33            | 0,14                   | 9,50           | 3,97                     |
| `ct::tests::div...`                                         | 0,26                  | 1,33            | 0,14                   | 9,50           | 3,97                     |
| `ct::tests::add...`                                         | 0,26                  | 1,33            | 0,14                   | 9,50           | 3,97                     |
| `ct::tests::mul...`                                         | 0,26                  | 1,32            | 0,14                   | 9,42           | 3,95                     |
| `frame::crypto::tests::try_fit_test...`                     | 0,66                  | 3,31            | 0,39                   | 8,48           | 3,78                     |
| `frame::stream::tests::try_fit_test...`                     | 0,67                  | 3,44            | 0,38                   | 9,05           | 3,91                     |
| `inet::checksum::tests::differential...`                    | 0,51                  | 2,46            | 0,26                   | 9,46           | 3,85                     |
| `inet::ipv4::tests::header_getter_setter_test...`           | 0,44                  | 2,11            | 0,22                   | 9,59           | 3,86                     |
| `inet::ipv4::tests::scope_test...`                          | 1,01                  | 5,18            | 0,56                   | 9,25           | 3,94                     |
| `inet::ipv6::tests::header_getter_setter_test...`           | 0,44                  | 2,17            | 0,23                   | 9,43           | 3,89                     |
| `inet::ipv6::tests::scope_test...`                          | 1,08                  | 5,34            | 0,59                   | 9,05           | 3,84                     |
| `interval_set::tests::interval_set_inset_range_test...`     | 0,83                  | 4,3             | 0,45                   | 9,55           | 4,00                     |
| `packet::number::map::tests::insert_value...`               | 0,28                  | 1,49            | 0,16                   | 9,31           | 4,02                     |
| `packet::number::sliding_window::test::insert_test...`      | 0,86                  | 4,22            | 0,46                   | 9,17           | 3,84                     |
| `packet::number::tests::example_test...`                    | 1,09                  | 5,67            | 0,59                   | 9,61           | 4,02                     |
| `packet::number::tests::rfc_differential_test...`           | 0,30                  | 1,62            | 0,18                   | 9,40           | 4.0                      |
| `packet::number::tests::truncate_expand_test...`            | 0,63                  | 3,23            | 0,35                   | 9,22           | 3,9                      |
| `packet::number::tests::round_trip...`                      | 0,22                  | 1,18            | 0,13                   | 9,00           | 4,00                     |
| `random::tests::gen_range_biased_test...`                   | 0,31                  | 1,58            | 0,18                   | 8,77           | 3,85                     |
| `recovery::rtt_estimator::test::weighted_average_test...`   | 0,43                  | 2,23            | 0,24                   | 9,29           | 3,97                     |
| `slice::tests::vectored_copy_fuzz_test...`                  | 0,73                  | 3,33            | 0,35                   | 9,51           | 3,75                     |
| `stream::iter::fuzz_target::fuzz_builder...`                | 0,26                  | 1,35            | 0,14                   | 9,64           | 4,02                     |
| `sync::cursor::tests::oracle_test...`                       | 0,92                  | 4,41            | 0,46                   | 9,58           | 3,86                     |
| `sync::spsc::tests::alloc_test...`                          | 0,82                  | 4,15            | 0,45                   | 9,22           | 3,91                     |
| `varint::tests::checked_ops_test...`                        | 0,95                  | 4,94            | 0,55                   | 8,98           | 3,92                     |
| `varint::tests::table_differential_test...`                 | 0,95                  | 4,93            | 0,51                   | 9,66           | 4,02                     |
| `varint::tests::eight_byte_sequence_test...`                | 0,96                  | 4,97            | 0,52                   | 9,55           | 4,00                     |
| `varint::tests::four_byte_sequence_test...`                 | 0,95                  | 4,95            | 0,52                   | 9,51           | 4,01                     |
| `varint::tests::two_byte_sequence_test...`                  | 0,97                  | 4,98            | 0,52                   | 9,57           | 3,99                     |
| `varint::tests::one_byte_sequence_test...`                  | 0,25                  | 1,32            | 0,14                   | 9,42           | 4,02                     |
| `varint::tests::round_trip_values_test...`                  | 0,27                  | 1,38            | 0,15                   | 9,20           | 3,92                     |
| `xdp::decoder::tests::decode_test...`                       | 0,58                  | 2,93            | 0,31                   | 9,45           | 3,94                     |
| `message::cmsg::tests::round_trip_test...`                  | 0,37                  | 1,59            | 0,19                   | 8,36           | 3,50                     |
| `message::cmsg::tests::iter_test...`                        | 0,77                  | 3,75            | 0,41                   | 9,14           | 3,83                     |
| `message::msg::tests::address_inverse_pair_test...`         | 0,85                  | 4,07            | 0,43                   | 9,46           | 3,84                     |
| `task::completion_to_tx::assign::tests::assignment_test...` | 0,36                  | 1,48            | 0,17                   | 8,70           | 3,47                     |
|                                                             | **Total**             | **Total**       | **Total**              | **Avg**        | **Avg**                  |
|                                                             | 22,8                  | 114,66          | 12,33                  | 9,29           | 3,90                     |

GOTO binary export is now the default because it is faster when there are multiple proof harnesses in a crate, but the switch `--write-json-symbtab` activates the JSON export if you need it. The memory consumption can be sometimes higher, but can also sometimes be lowe depending on how much opportunity for sharing of identical subtrees the crate offers.

## Details on the GOTO binary format

Internally, CBMC uses a single generic data structure called [`irept`](https://github.com/diffblue/cbmc/blob/develop/src/util/irep.h#L308) to represent all tree-structured data (see [statements](https://github.com/diffblue/cbmc/blob/develop/src/util/std_expr.h), [expressions](https://github.com/diffblue/cbmc/blob/develop/src/util/std_expr.h), [types](https://github.com/diffblue/cbmc/blob/develop/src/util/std_types.h) and [source locations](https://github.com/diffblue/cbmc/blob/develop/src/util/source_location.h) in the CBMC code base). A GOTO binary is mainly a collection of serialised `irept`.

The `Irep` type would look like this if written in Rust:

```rust
// an opaque type for an interned String
struct IrepId;

// a generic tree node
struct Irep {
    // node identifier, defines the interpretation of the node
    id: IrepId,
    // Subtrees indexed by integer
    sub: Vec<Rc<Irep>>,
    // Subtrees keyed by name
    named_sub: Map<IrepId, Rc<Irep>>,
}
```

`Ireps` are tagged by an `IrepId` (an interned string) giving them their meaning. An `Irep` references other `Ireps` through reference counted smart pointers. `Ireps` also allow safe sharing of subtrees, with a copy-on-write update mechanism. For instance the expression `x + y` would be represented by an `Irep` similar to this:

```rust
Irep {
    id = IrepId("+"),
    sub: Vec(
        Irep(
            id = "symbol_expr",
            named_sub: Map((IrepId("identifier"), Irep(id: IrepId("x"))))
        ),
        Irep(
            id = "symbol_expr",
            named_sub: Map((IrepId("identifier"), Irep(id: IrepId("y"))))
        ),
    )
}
```

The serialization/deserialization algorithm for GOTO binaries uses a technique called *value numbering* to avoid repeating identical `Ireps` and strings in the binary file, and uses *7-bit variable length encoding* for integer values to reduce the overall file size.

A *value numbering function* for a type `K` is a function that assigns a unique number in the range `[0, N)` to each value in a multiset `S` of values of type `K` ((multiset: some values can be repeated).  Numberings are usually implemented using a hash map of type `HashMap<K, usize>`. Each value 'k' in the set `S` is numbered by performing a lookup in the map: if an entry for `k` is found, return the associated value, otherwise insert a new entry `(k, numbering.size())` and return the unique number for that entry. Value numbering for `Ireps` uses vectors of integers built from the unique numbers of the `Irep` fields as keys. An `Irep` is uniquely numbered by recursively numbering its id, subtrees and named subtrees, assembling a key from these unique numbers and looking up the unique number for that key in the numbering. Since two `Ireps` with the same id, subtrees and named subtrees are represented by the same key, the unique number of the key identifies the `Irep` by its contents. CBMC's binary serde algorithm uses numbering functions for `Ireps` and `Strings` that are used as a cache of already serialised Ireps and Strings. An `Irep` node is fully serialised only the first time it is encountered. Later occurrences are serialised by reference, i.e. only by writing their unique identifier. This achieves maximum sharing of identical subtrees and strings in the binary file.

The *7-bit variable length encoding* scheme for integers works as follows: an integer of `N` bytes is serialised to a list of `M` bytes, where each byte encodes a group of seven bits of the original integer and one bit signals the continuation or termination of the list. For instance, the decimal value 32bit decimal `190341` is represented as `00000000000000101110011110000101` in binary. Splitting this number in groups of 7-bits starting from the right, we get `0000101 1001111 0001011 0000000 0000`. We see that all bits in the two last groups are false, so only the first three groups will be serialised. With continuation bits added (represented in parentheses), the encoding for this 4-byte number only uses 3-bytes:  `(1)0000101(1)1001111(0)0001011`.

## Making Constant Propagation Field-Sensitive for Union Types in CBMC

Union types are very common in goto-programs emitted by Kani, due to the fact that Rust code typically uses `enums`, which are themselves modelled as tagged unions at the goto-program level. Initially the *field sensitivity* transform in CBMC enabled constant propagation for individual array cells and in individual struct fields, but not for union fields. Constant propagation helps pruning control flow branches during symbolic execution and can greatly reduce the runtime of an analysis. Ensuring that constant propagation also works for union fields is important for Rust programs. We extended field-sensitivity  to union fields  in [cbmc-5.71.0](https://github.com/diffblue/cbmc/releases/tag/cbmc-5.71.0), [PR 7230](https://github.com/diffblue/cbmc/pull/7230). This vastly improved performance for Rust programs manipulating  `Vec<T>`  data types, and allowed us to solve a number performance issues reported by our users: [#705](https://github.com/model-checking/kani/issues/705)  [#1226](https://github.com/model-checking/kani/issues/1226)  [#1657](https://github.com/model-checking/kani/issues/1657)  [#1673](https://github.com/model-checking/kani/issues/1673)  [#1676](https://github.com/model-checking/kani/issues/1676).

### The Field Sensitivity Transform

CBMC's constant propagation algorithm only propagates values for scalar variables with a basic datatype such as `bool`, `int`, ... but not for aggregates. To enable propagation for individual fields of aggregates, CBMC decomposes them into their individual scalar fields, by introducing new variables in the program, and resolving all field access expressions to these new variables.

For unions, just like for structs, the transform introduces a distinct variable for each field of the union. However, contrary to structs, the different fields of a union overlap in the byte-level layout. As a result, every time a union field gets assigned with a new value, all fields of the union are actually impacted, and the impact depends on how the fields overlap in the layout. This means that the variables representing the different fields of the union have to be handled as a group and globally updated after each update to any one of them.

For instance, for a union defined as follows (using C syntax for simplicity):

```c
union {
  unsigned long a;
  unsigned int b;
} u;
```

The byte-level layout of the fields is such that the lowest 4 bytes of `u.a` and all bytes of `u.b` overlap. As a result, updating `u.a` updates all bytes `u.b`, and updating `u.b` updates the lowest 4 bytes of `u.a`:

```
u       uuuuuuuuuuuuuuuu
u.a     aaaaaaaaaaaaaaaa
u.b             bbbbbbbb
idx.   16       7      0
      MSB             LSB
```

The transform introduces a variable `u_a` to represent `u.a`, and a variable `u_b` to represent `u.b`. These two variables are not independent, and every-time one of the fields is updated in the original program, both variables are updated in the transformed program. 

Applying the transform to the following program:

```c
int main() {
  union {
    unsigned long a;
    unsigned int b;
  } u;

  u.a = 0x0000000000000000;
  u.b = 0x87654321;
  assert(u.a == 0x0000000087654321);
  assert(u.b == 0x87654321);
  return 0;
}
```

produces the following transformed program:

```c
int main() {
  unsigned long u_a;
  unsigned int u_b;

  // the bytes of u_b are equal the low bytes of u_a
  u_b = (unsigned int) u_a;

  // u.a = 0x0000000000000000;
  u_a = 0x0000000000000000;
  u_b = (unsigned int) 0x0000000000000000;

  // u.b = 0x87654321;
  u_a = (u_a & 0xFFFFFFFF00000000) | ((unsigned long) 0x87654321);
  u_b = 0x87654321;

  // assert(u.a == 0x0000000087654321);
  assert(u_a == 0x0000000087654321);

  // assert(u.b == 0x87654321);
  assert(u_b == 0x87654321);
  return 0;
}
```

# Conclusion

The cumulative effects of these changes provides for a better user experience, faster code-generation times, greatly reduced peak memory consumption, and better analysis performance.

[^1]: Without our enhancement to CBMC, it was already possible to select a different SAT solver without rebuilding CBMC via the `--external-sat-solver` option. However, this option doesn't use the solver in an incremental fashion through its library API, and instead relies on writing DIMACS files to disk, which often results in subpar performance.

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

# Allowing SAT Solver Selection from Harnesses

SAT solving is typically the most time-consuming part of a Kani run. There is a large number of SAT solvers available, whose performance can vary widely depending on the specific type of formula, and it can be very helpful to be able to try different SAT solvers on the same problem to dertermine which one performs best.

By default, CBMC uses the [MiniSat](http://minisat.se/) SAT solver, but it also allows to use a different SAT (or [SMT](https://en.wikipedia.org/wiki/Satisfiability_modulo_theories)) through a command line switch.

To ease the selection of the SAT solver for Kani users, we've introduced a Kani attribute, `kani::solver`, that can be used to specify the SAT solver to use for each harness.

For instance, one can configure Kani to use [CaDiCaL](https://github.com/arminbiere/cadical) as follows:

```
#[kani::proof]
#[kani::solver(cadical)] // <--- Use CaDiCaL for this harness
fn my_harness() { ... }
```

Changing the solver can result in orders of magnitude performance difference. Thus, we encourage users to try different solvers to find the one that performs the best on their harness. At the time of writing, the following three solvers are supported out of the box by Kani: `minisat` (the default solver), `cadical`, and `kissat` ([Kissat](https://github.com/arminbiere/kissat)). Kani also allows using other SAT solvers available as standalone binaries in your system `PATH`. This can be done using:

```
`#[kani::solver(bin="<SAT_SOLVER_BINARY>")]                                                                                 `
```

An example of a SAT solver that we've found effective for certain classes of programs (e.g. ones involving cryptographic operations) is [CryptoMiniSat](https://github.com/msoos/cryptominisat). After installing CryptoMiniSat and adding the binary to your path, you can configure Kani to use it via:

```
#[kani::solver(bin="cryptominisat5")]
```

As illustrated by this [this pull-request](https://github.com/aws/s2n-quic/pull/1771) on the [s2n-quic repository](https://github.com/aws/s2n-quic), using `cadical` instead of the default minisat brought speedups in the range of x1.5 x28. And all it takes to switch is adding an attribute to your harness!

# Adding Direct Export of GOTO Binaries

The Kani compiler translates a Rust program into a *symbol table* expressed in the format expected by CBMC. A symbol table lists all symbols manipulated by the program together with their source locations and definitions (types symbols and their definitions, functions symbols and their bodies, static symbols and their initial value, etc.) represented as abstract syntax trees.

Kani originally serialized symbol tables to JSON and used the CBMC utility `symtab2gb`  to translate the JSON symbol table to GOTO binary format, which is the format that CBMC actually expects as input. The JSON to GOTO binary conversion was one of the most time and memory consuming steps in the compilation stage.

We implemented direct GOTO binary serialisation in Kani (see code [here](https://github.com/model-checking/kani/blob/main/cprover_bindings/src/irep/goto_binary_serde.rs)), which allows us to skip the costly invocation of `symtab2gb`. Kani now exports GOTO symbol tables 2x-4x faster with a 2x-3x reduction in peak memory consumption.

The table below reports  time and memory consumption to parse and compile two commonly used crates to JSON vs GOTO binaries.

TODO benchcomp numbers here
- `kani --write-json-symtab` vs `kani` (using the kani 0.32 and cbmc 5.88.0), isolating the the codegen + goto model export time.

This new export method is globally faster and more memory efficient and has become the new default.


## Details on GOTO binary format

Internally, CBMC uses a single generic data structure called [`irept`](https://github.com/diffblue/cbmc/blob/develop/src/util/irep.h#L308) to represent all tree-structured data (see [statements](https://github.com/diffblue/cbmc/blob/develop/src/util/std_expr.h), [expressions](https://github.com/diffblue/cbmc/blob/develop/src/util/std_expr.h), [types](https://github.com/diffblue/cbmc/blob/develop/src/util/std_types.h) and [source locations](https://github.com/diffblue/cbmc/blob/develop/src/util/source_location.h) in the CBMC code base). 

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

A GOTO binary mostly contains serialised `Ireps`. The serde algorithm for GOTO binaries uses *value numbering* to avoid repeating identical `Ireps` and strings in the binary file, and uses *7-bit variable length encoding* for integer values to reduce overall file size.

A *value numbering function* for a key type `K` is a function that assigns a unique number in the range `[0, N)` to each value in a set of values of type `K`. Numberings are usually implemented using a hash map of type `HashMap<K, usize>`. Each value of the set is numbered by performing a lookup in the map: if an entry for `k` is found, return the associated value, otherwise insert a new entry `(k, numbering.size())` in the numbering map and return the unique number for that entry. Value numbering for `Ireps` uses vectors of integers built from the unique numbers of the irep fields as keys. An `Irep` is uniquely numbered by recursively numbering its id, subtrees and named subtrees, then assembling a key from these unique numbers and looking up the unique number for that key. Since two `Ireps` with the same id, subtrees and named subtrees are represented by the same key, the unique number of the key identifies the `Irep` by its contents. CBMC's binary serde algorithm uses numbering functions for `Ireps` and `Strings` that are used as a cache of already serialised Ireps and Strings. An `Irep` node is fully serialised only the first time it is encountered. Later occurrences are serialised by reference, using their unique identifier. This achieves maximum sharing of identical subtrees and strings in the binary file.

The *7-bit variable length encoding* scheme for integers works as follows: *a*n integer is serialised to a list of 8-bits words, where each 8-bit word encodes a group of seven bits of the original integer and one bit signals the continuation or termination of the list. For instance, the decimal value 32bit decimal `190341` is represented as `00000000000000101110011110000101` in binary. Splitting it in groups of 7-bits starting from the right, we get `0000101 1001111 0001011 0000000 0000`. We see that all bits in the two last groups are false, so only the first three groups are serialised. With continuation bits added (represented in parentheses), the encoding for this 32 bit number only uses 24 bits:  `(1)0000101(1)1001111(0)0001011`.

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
# fsharp-wasm-sci

<!-- status -->
[Status of every repository](https://fsharp-wasm.github.io/status/)
<!-- /status -->

<!-- platform -->
> **Part of the [fsharp-wasm platform](https://fsharp-wasm.github.io/projects/)**: scientific computing. Will build on [core](https://github.com/fsharp-wasm/fsharp-wasm-core). Status: **planned, no code yet**.
> [All projects](https://fsharp-wasm.github.io/projects/) · [Roadmap](https://fsharp-wasm.github.io/docs/roadmap/) · [Tests](https://fsharp-wasm.github.io/tests/) · [Benchmarks](https://fsharp-wasm.github.io/benchmarks/) · [Live demos](https://fsharp-wasm.github.io/demos/)

Scientific computing for F# that runs as WebAssembly GC, in the browser and on the edge: linear algebra, geospatial analysis, chemistry and quantum simulation, written in F# with no
native dependency, so that a program carries only what it calls.

**This repository is a plan, not yet a library.** Nothing here compiles. What exists is the layer it will stand on, and it lives where every other library can reach it
([the rule](https://github.com/fsharp-wasm/fsharp-wasm/blob/develop/AGENTS.md): code goes in the lowest layer all its users can reach).

## What is there to build on

| | where | state |
|---|---|---|
| hierarchies of ordered leaves, a cube of prefix sums over two of them (the total of any group over any group is four reads), pivoting, drill-down, formulas over measures | [`FSharp.Wasm.Frame`](https://github.com/fsharp-wasm/fsharp-wasm-core), used by [the accounting grid](https://github.com/fsharp-wasm/fsharp-wasm-accounting) | working; a million lines turned in 4 to 5 ms (measured 2026-11-24, a headless Chromium) |
| JSON and numbers | `FSharp.Wasm.Data` in core | working |
| flat tensors with strides, ragged arrays as offset buffers, columnar tables with group-by, join and radix sort | planned for core, next to `Frame`: the accounting grid and these libraries need the same thing underneath | not started |
| the test framework, with differential tests against .NET | [fsharp-wasmecto](https://github.com/fsharp-wasm/fsharp-wasmecto) | working |
| the benchmark harness for size and speed | [fsharp-wasm-runtime](https://github.com/fsharp-wasm/fsharp-wasm-runtime) | working |

## What will be here

Libraries, in this order, each only when the layer below it is real and measured:

1. **Linear algebra** (`FSharp.Sci.Linalg`): blocked matrix multiplication, LU, Cholesky, QR, compressed sparse rows.
2. **Geospatial** (`FSharp.Sci.Geo`): flat-buffer spatial indexes, WGS84 and Web Mercator and UTM, great-circle distances, polygon area and clipping, raster grids.
3. **Chemistry** (`FSharp.Sci.Chem`): a periodic table as compile-time constants, molecular graphs, a SMILES parser, masses and formulas.
4. **Quantum** (`FSharp.Sci.Quantum`): a complex number without allocation, a state-vector simulator, the usual gates.

Where a workload is too large for F# in a browser, an optional Rust library behind a WIT interface is planned (the same Rust is not written twice); the F# version stays the default, because it is small.

## Principles

- **Memory layout first**: a dense tensor is one flat array and its strides, a ragged array is its values and offsets, so that the garbage collector has few objects and the loops are cache friendly.
- **Source only, and only what is called**: the compiler emits the functions and types a program uses.
- **Every number is measured and dated**: size, speed against .NET and against JavaScript, and agreement with a reference (differential tests). A target is not a claim.

## Contributing

Open an issue for the library you want (the [roadmap](https://fsharp-wasm.github.io/docs/roadmap/) and the issues of this repository say what is next), and read
[AGENTS.md](AGENTS.md) for how the platform is worked on. Code is Apache-2.0.

## License

[Apache-2.0](LICENSE).

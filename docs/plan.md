# Plan for fsharp-wasm-sci

Written 2026-11-26. Low priority: nothing here is started, and nothing is decided that a measurement has not decided. This page says what to build in which order, and the one question
that has to be answered with numbers before the first library is written (how data crosses a boundary).

## What the libraries are for

The first customers are already in the platform: the accounting grid (`FSharp.Wasm.Frame`: cubes of prefix sums, pivoting, formulas), the dashboards (`FSharp.Wasm.Telemetry`: samples, a historian
that keeps a history small) and the charts (millions of points). They need the same thing underneath that science needs: **dense numbers in flat arrays, with their shape**. So the order is from
the bottom, and the first piece is not in this repository.

## Order

| step | what | where | done when |
|---|---|---|---|
| 0 | flat tensors with strides, ragged arrays as offset buffers, columnar tables with group-by, join and radix sort | fsharp-wasm-core, next to `Frame` (task: tensors, ragged arrays and tables) | tests under .NET and as WebAssembly GC, differential against .NET; a measured size and speed |
| 1 | `FSharp.Sci.Linalg`: blocked matrix multiplication, LU, Cholesky, QR, compressed sparse rows | here | the same, plus agreement with a reference to a stated tolerance |
| 2 | `FSharp.Sci.Geo`: flat-buffer spatial indexes, WGS84, Web Mercator, UTM, great-circle distances, polygon area and clipping, raster grids | here | |
| 3 | `FSharp.Sci.Chem`: a periodic table as constants, molecular graphs, a SMILES parser, masses and formulas | here | |
| 4 | `FSharp.Sci.Quantum`: a complex number that does not allocate, a state-vector simulator, the usual gates | here | |

A step is done when its numbers are in the README with a date, as in the other repositories: size, speed against .NET and against JavaScript, agreement with a reference.

## The question before step 1: how does data cross a boundary

The README says "an optional Rust library behind a WIT interface". That was written before the boundary was measured, and it should not stand as a decision. What is known:

- A WebAssembly GC module has **no linear memory** that anyone else can see: its arrays are GC objects. A Rust or C library compiled to WebAssembly has a linear memory. They cannot share a buffer.
- WIT with the canonical ABI **copies** across that line. The spike in fsharp-wasm-edge (`docs/spikes/wit-glue`, 2026-11) made a GC module a component with a generated adapter, and measured the copy:
  4 characters 10 µs a call, 64 KB 2.1 ms, 2 MB 49 ms. For a call that moves a kilobyte that is nothing; for a matrix of a million numbers on every call it is the whole cost.
- What is not measured: records, lists of records, `option`, `result`, resources; another host than jco on Node; a Rust library inside a GC component.

Three ways, to be put side by side:

1. **F# only, in GC arrays** (the default and the baseline). No boundary. Slower where SIMD or threads would help, and the compiler has neither yet (no `v128`, no shared GC arrays).
2. **A kernel module beside it, with handles instead of data.** A library of kernels (Rust, C, or F# compiled for linear memory) owns the buffers, as a GPU does: the F# side holds *handles* (an id, a shape, a type), and an operation (`matmul a b`) is a call with three integers. Data crosses only at the ends (`upload`, `download`), once. The contract is a small set of coarse operations, which is what WIT (or a function table) describes well. Zero copy between operations; a copy at the ends.
3. **WIT with the data in the signature** (`list<f32>`): the portable form, any host that speaks the Component Model. A copy on every call.

My view, to be checked: **2 is the better design for numbers, and WIT belongs to the control plane only** (the list of operations, so that a non-JavaScript host such as wasmtime can embed a kernel), not to the data. 1 stays the default
because a program that calls no kernel carries none. This is a view and not a result.

### The spike that decides it

Before `FSharp.Sci.Linalg` is written: one operation, `matmul` of f32 matrices (64², 256², 1024²), done four ways, each measured in the same headless browser and under Node, with the size of what
ships and the time of the first call:

| | |
|---|---|
| a | F# in GC arrays (blocked, a plain loop) |
| b | F# in GC arrays with the loop the compiler emits `v128` for, when that exists (otherwise: stated as not possible) |
| c | a Rust kernel in linear memory with SIMD, handles in, handle out (design 2) |
| d | the same kernel through WIT with `list<f32>` in and out (design 3) |

The rule that decides: if (a) is within a factor of 3 of (c) at 256², the kernel is not worth its weight and design 1 stays; if (c) beats (d) by more than the cost of one upload, design 2 is the contract;
if (d) is within 10 % of (c), WIT is simple enough to be the whole contract. The numbers go into this file with their date.

## What the compiler has to do first

Found by the other libraries, and to be fixed in the compiler before a library works around them (the platform rule):

- `Math.Sin`, `Cos`, `Tan`, `Atan2` and `Exp` are accurate to 1e-14 since 2026-11-26 (they were 1e-5 to 1e-7): a linear algebra or quantum library needs that. `Math.Pow`, `Log`, `Sqrt` of the helpers have not been measured the same way.
- `sprintf "%A"` of a record, a union, a list or a tuple does not compile (a printing of a tensor wants it, a test failure message wants it).
- A `Complex` that is a struct without an allocation, and arrays of structs that are flat: not measured.
- `v128` (SIMD) from `System.Numerics.Vector<T>`: not there.
- Generic local functions that are used at another type than `int` (STATUS.md, known gaps).

## Not in the plan

- Python's names. `np.dot` is not a goal; the names are F#'s, and a function that takes a tensor and a function is the style.
- GPU. WebGPU is used by the canvas host (SDF instances); a compute path through WebGPU is a later question for the kernels of design 2 and is not planned here.
- Threads. WebAssembly threads need shared linear memory; a GC array cannot be shared yet.

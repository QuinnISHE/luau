# Luau Compiler / Runtime Performance Explorer — Orchestrator Spec

**Date:** 2026-05-30  
**Audience:** autonomous Luau source explorers, compiler readers, benchmark agents, and documentation agents  
**Goal:** produce a source-grounded, benchmark-validated set of markdown documents explaining how Luau optimizes code, what gets folded, what becomes fast bytecode or native code, what falls back, and what source-code shapes should be preferred or avoided in high-performance Luau.

This package is intentionally **general Luau research**, not a project-specific physics-engine spec. Examples may use math-heavy kernels because they stress the optimizer well, but the final conclusions must apply broadly to Luau code.

---

## One-sentence task for the explorers

> Clone the Luau repository, inspect the compiler, bytecode, VM, builtin, vector, buffer, and native-codegen implementation, validate findings with bytecode/IR/native/profiler experiments, and output a multi-document Luau performance manual that explains exactly which source-code patterns optimize well, which patterns block optimization, and why.

---

## Required final output documents

The explorers must produce these markdown files in the final research bundle:

```text
01_Luau_Compiler_Pipeline_Map.md
02_Bytecode_Optimization_And_Folding_Matrix.md
03_Builtins_Fastcalls_Vector_Buffer_Report.md
04_Native_CodeGen_IR_And_Lowering_Report.md
05_Type_Annotations_Table_Shapes_And_AntiPatterns.md
06_Benchmark_Harness_And_Results.md
07_Luau_Performance_Rules_And_Skill_Draft.md
08_Open_Questions_And_Unverified_Claims.md
```

Each document must be useful on its own. The final `07_Luau_Performance_Rules_And_Skill_Draft.md` should be concise enough to turn into a reusable agent skill or memory, while the earlier documents should preserve the detailed evidence.

---

## Source priority rules

Use sources in this order:

1. **Luau repository source code** at the exact commit hash used by the explorers.
2. **Official Luau documentation**.
3. **Roblox Creator documentation**, especially for Studio/native-codegen behavior.
4. **Luau RFCs**.
5. **Luau release notes**.
6. Community posts only when they clarify a known behavior that is also verified in source or by benchmark.

Do not accept a performance claim unless it is marked as one of:

| Label | Meaning |
|---|---|
| `SOURCE_VERIFIED` | Verified in Luau source with file path, function/class name, and commit hash. |
| `DOC_VERIFIED` | Supported by official Luau/Roblox docs or Luau RFCs. |
| `BENCH_VERIFIED` | Reproduced by a local benchmark or profiler trace. |
| `INFERRED` | Logically inferred from source/docs but not benchmarked yet. |
| `UNVERIFIED` | Plausible but not proven; must not appear as a final rule. |
| `ROBLOX_RUNTIME_DEPENDENT` | True in Roblox Studio/runtime behavior, but not necessarily derivable from open-source Luau alone. |

Every final recommendation must be either `SOURCE_VERIFIED`, `DOC_VERIFIED`, `BENCH_VERIFIED`, or explicitly marked as a hypothesis.

---

## Required source snapshot metadata

At the top of every output document, include:

```text
Luau repo: https://github.com/luau-lang/luau
Luau commit: <full git hash>
Luau release/tag if applicable: <tag or none>
Roblox Studio version if benchmarked in Studio: <version>
Operating system: <OS + version>
CPU architecture: <x64/arm64 + CPU model>
Benchmark date: <YYYY-MM-DD>
Native codegen availability: <yes/no/platform/details>
Compiler flags tested: <O0/O1/O2/debug-level/codegen/native/etc.>
```

---

## Baseline official references

Explorers must read and cite these sources in the final bundle:

- Luau performance overview: `https://luau.org/performance/`
- Roblox native code generation docs: `https://github.com/Roblox/creator-docs/blob/main/content/en-us/luau/native-code-gen.md`
- Luau vector library RFC: `https://rfcs.luau.org/vector-library.html`
- Luau type system overview: `https://luau.org/types/`
- Luau syntax reference: `https://luau.org/syntax/`
- Luau repository: `https://github.com/luau-lang/luau`
- Luau bytecode header: `Common/include/Luau/Bytecode.h`
- Luau release notes: `https://github.com/luau-lang/luau/releases`

These are starting points, not the complete bibliography.

---

## High-level research questions

The explorers must answer all of these:

1. What is the full source-to-bytecode pipeline?
2. Which optimizations happen in the AST/compiler frontend?
3. Which optimizations happen during bytecode generation?
4. Which optimizations happen in the VM/interpreter at runtime?
5. Which optimizations happen in native codegen IR and lowering?
6. Which operations are core bytecode instructions versus builtin fastcalls versus normal calls?
7. Which builtins are constant-folded, and at which optimization level?
8. Which builtins are fastcalled but not folded?
9. Which operations are optimized only when call shape is obvious to the compiler?
10. How do `--!optimize`, `--!native`, `@native`, `--!strict`, and type annotations affect code generation?
11. How do `vector`, `Vector3`, `buffer`, `bit32`, `math`, `string`, `table`, and globals behave under bytecode/native optimization?
12. Which source-code patterns deoptimize or fall back?
13. Which source-code patterns allocate unexpectedly?
14. Which table shapes and field-access patterns preserve inline caching?
15. Which local-function/inlining patterns get optimized?
16. Which closures are cached or cheap, and when are they still allocation-heavy?
17. What should a high-performance Luau code-shape guide recommend?
18. Which claims remain unknown because Roblox runtime behavior differs from open-source Luau?

---

## Non-goals

Do not spend time on:

- General Lua style advice unless it affects Luau optimization.
- Project-specific algorithms.
- Micro-optimizations that cannot be tied to source, docs, bytecode, native output, or benchmark data.
- Recommendations that only improve synthetic benchmarks but hurt readability and are not relevant to hot paths.
- Guessing Roblox internal behavior that is not exposed in docs or measurable in Studio.

---

## Output quality bar

A good final rule looks like this:

```markdown
### Prefer direct builtin calls or localized builtins over indirect calls

**Rule:** Prefer `math.max(x, y)` or `local max = math.max; max(x, y)` in hot code. Avoid hiding builtins behind dynamic table fields or higher-order wrappers when you need fastcall behavior.

**Why:** Luau's fastcall mechanism requires the call to be obvious to the compiler. Direct builtin calls and localized builtin calls qualify; indirect calls do not unless inlined.

**Evidence:**
- DOC_VERIFIED: `https://luau.org/performance/`, Specialized builtin function calls.
- SOURCE_VERIFIED: `<commit> Compiler/src/Builtins.cpp`, `<function name>`.
- BENCH_VERIFIED: `bench_fastcall_direct_vs_indirect.luau`, median direct call 1.00x baseline, indirect 1.43x slower on <machine>.

**Counterexamples / caveats:** `obj:Method()` has a separate namecall optimization; do not replace it with cached method calls blindly.
```

A bad final rule looks like this:

```markdown
Use locals because locals are faster.
```

That is too broad and misses Luau-specific import/global and method-call behavior.

---

## Explorer roles

Run these roles separately when possible:

### 1. Compiler / bytecode explorer

Reads `Compiler/`, `Common/include/Luau/Bytecode.h`, bytecode tests, compile CLI tools, and release notes. Produces documents 01 and 02.

### 2. Builtin / VM / vector / buffer explorer

Reads builtin registration, builtin folding, fastcall implementation, vector and buffer support, VM interpreter paths, and relevant tests. Produces document 03.

### 3. Native-codegen explorer

Reads `CodeGen/`, IR utils, IR passes, lowering, native builtins, type specialization, VM exits, and code-size limits. Produces document 04.

### 4. Code-shape and anti-pattern explorer

Reads performance docs, source implementation, and benchmark results to produce practical code guidance. Produces document 05.

### 5. Benchmark explorer

Builds and runs repeatable bytecode/native/profiler benchmarks. Produces document 06.

### 6. Documentation synthesizer

Condenses all evidence into a final reusable guide and skill draft. Produces documents 07 and 08.

---

## Acceptance gates

The final bundle is not acceptable until:

- Every performance rule has evidence labels.
- Every benchmark includes source code, run command, environment, and raw results.
- Every “avoid” recommendation includes at least one counterexample or caveat.
- Every compiler behavior claim names the relevant source file and function/class.
- Every optimization-level claim is tested at multiple optimization levels.
- Native-codegen claims separate open-source Luau behavior from Roblox Studio behavior.
- Unverified or ambiguous findings are isolated in `08_Open_Questions_And_Unverified_Claims.md`.

---

## Final deliverable layout

Use this folder layout:

```text
luau-performance-explorer-output/
  README.md
  01_Luau_Compiler_Pipeline_Map.md
  02_Bytecode_Optimization_And_Folding_Matrix.md
  03_Builtins_Fastcalls_Vector_Buffer_Report.md
  04_Native_CodeGen_IR_And_Lowering_Report.md
  05_Type_Annotations_Table_Shapes_And_AntiPatterns.md
  06_Benchmark_Harness_And_Results.md
  07_Luau_Performance_Rules_And_Skill_Draft.md
  08_Open_Questions_And_Unverified_Claims.md
  benches/
    *.luau
    results.csv
    bytecode_dumps/
    native_code_size_dumps/
  source_notes/
    source_file_index.md
    commit_diff_notes.md
```

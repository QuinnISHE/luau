# Final Output Templates and Skill Draft Spec

This document defines the final documentation format for the Luau performance research bundle.

---

## Required final bundle

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
```

---

## README template

```markdown
# Luau Performance Explorer Output

## Scope

This bundle documents Luau compiler, VM, builtin, vector, buffer, and native-codegen optimizations as of:

- Luau commit:
- Roblox Studio version:
- Date:

## How to read this bundle

Start with `07_Luau_Performance_Rules_And_Skill_Draft.md` for practical rules. Use documents 01-06 for evidence.

## Evidence labels

<repeat evidence-label table>

## Top 10 findings

1. TBD
2. TBD
...

## Top 10 anti-patterns

1. TBD
2. TBD
...

## Open questions

See `08_Open_Questions_And_Unverified_Claims.md`.
```

---

## `01_Luau_Compiler_Pipeline_Map.md` template

```markdown
# Luau Compiler Pipeline Map

## Source snapshot metadata

## Executive summary

## Pipeline diagram

## Stage-by-stage source map

| Stage | Source files | Key functions/classes | Inputs | Outputs | Optimizations |
|---|---|---|---|---|---|

## Optimization levels

| Level | Meaning | Enabled optimizations | Disabled optimizations | Evidence |
|---|---|---|---|---|

## Important source files

## Notes for future compiler changes

## Open questions
```

---

## `02_Bytecode_Optimization_And_Folding_Matrix.md` template

```markdown
# Bytecode Optimization and Folding Matrix

## Source snapshot metadata

## Summary table

| Source pattern | Folds? | O-level | Bytecode before | Bytecode after | Source proof | Caveat |
|---|---:|---|---|---|---|---|

## Numeric folding

## Boolean/control folding

## String folding

## Vector folding

## Builtin folding

## Table-field propagation

## Import/global chain optimization

## Inlining/loop-unrolling interaction

## Non-folded cases

## Practical rules

## Open questions
```

---

## `03_Builtins_Fastcalls_Vector_Buffer_Report.md` template

```markdown
# Builtins, Fastcalls, Vector, and Buffer Report

## Source snapshot metadata

## Builtin taxonomy

| Builtin/op | Core op | Fastcall | Folded | Native lowered | Fallback | Evidence |
|---|---:|---:|---:|---:|---|---|

## Fastcall mechanism

## Math

## Bit32

## String

## Table

## Buffer

## Vector

## Vector/scalar break-even results

## Practical rules

## Open questions
```

---

## `04_Native_CodeGen_IR_And_Lowering_Report.md` template

```markdown
# Native CodeGen IR and Lowering Report

## Source snapshot metadata

## Native-codegen model

## IR pipeline

## IR passes

## Type specialization

## Guards and VM exits

## Lowering matrix

| Source pattern | IR | Native lowering | Guards/exits | Evidence | Caveats |
|---|---|---|---|---|---|

## Roblox Studio validation

## Code size limits

## Failure modes

## Practical rules

## Open questions
```

---

## `05_Type_Annotations_Table_Shapes_And_AntiPatterns.md` template

```markdown
# Type Annotations, Table Shapes, and Anti-Patterns

## Source snapshot metadata

## Practical do/avoid table

| Topic | Prefer | Avoid | Why | Evidence | Caveat |
|---|---|---|---|---|---|

## Type annotations

## Table shapes

## Metatables and methods

## Globals and environment purity

## Functions, closures, upvalues

## Loops and iteration

## Allocation and GC

## Buffer code shapes

## Vector code shapes

## Native-codegen anti-patterns

## Open questions
```

---

## `06_Benchmark_Harness_And_Results.md` template

```markdown
# Benchmark Harness and Results

## Source snapshot metadata

## Benchmark methodology

## Environment

## Raw result files

## Benchmark family summaries

## Result tables

## Bytecode excerpts

## Native/profiler excerpts

## Benchmark-backed rules summary

| Rule | Confidence | Evidence | Caveat |
|---|---|---|---|

## Rejected/noisy benchmarks

## Open questions
```

---

## `07_Luau_Performance_Rules_And_Skill_Draft.md` template

This is the document that should be turned into a reusable skill or Muninn memory. It must be shorter than the evidence docs but still precise.

```markdown
# Luau Performance Rules and Skill Draft

## Purpose

Use this when writing or reviewing performance-sensitive Luau. Prefer readable code unless a rule applies to hot paths.

## Confidence legend

- **Hard rule:** source/docs + benchmark agree.
- **Benchmark rule:** measured in tested environment; may require revalidation after Luau updates.
- **Hypothesis:** plausible but not yet proven; do not enforce automatically.

## Hot-path checklist

1. Is this code actually hot according to profiler?
2. Is the code direct Luau computation rather than Roblox API bound?
3. Are native/codegen/code-size constraints relevant?
4. Are allocations avoided in tight loops?
5. Are builtin calls obvious to the compiler?
6. Are table shapes stable?
7. Are type annotations present where inference is ambiguous?
8. Are vector constructions amortized?
9. Are buffer reads/writes shaped well?
10. Are claims benchmarked on the target runtime?

## Rules

### Rule 1: <name>

**Use when:**
**Prefer:**
```luau
```
**Avoid:**
```luau
```
**Why:**
**Evidence:**
**Caveat:**

### Rule 2: <name>
...

## Vector/scalar decision matrix

| Situation | Prefer | Reason | Confidence |
|---|---|---|---|

## Builtin call decision matrix

| Situation | Prefer | Avoid | Reason |
|---|---|---|---|

## Native-codegen decision matrix

| Situation | Use `--!native`? | Use `@native`? | Avoid native? | Reason |
|---|---:|---:|---:|---|

## Anti-patterns

## Benchmark recipes

## Update protocol after Luau changes
```

---

## `08_Open_Questions_And_Unverified_Claims.md` template

```markdown
# Open Questions and Unverified Claims

## Source snapshot metadata

## Unverified claims

| Claim | Why it matters | What evidence is missing | How to test | Owner |
|---|---|---|---|---|

## Conflicting findings

| Topic | Source A | Source B | Benchmark | Current decision |
|---|---|---|---|---|

## Roblox-runtime-dependent questions

## Future Luau release watchlist

## Claims explicitly rejected
```

---

## Final skill quality rules

The skill draft must:

- Prefer general Luau guidance over project-specific guidance.
- Use short code examples.
- Never state a benchmark-specific rule as universal without caveats.
- Clearly separate interpreter, bytecode, and native-codegen rules.
- Warn that Luau evolves and rules must be revalidated after major releases.
- Include a “measure first” rule for any optimization that reduces readability.
- Include a “don’t cargo-cult Lua folklore” rule for common Lua/LuaJIT advice that is different in Luau.

---

## Minimum final practical rules to include if verified

The final skill should include rules for these topics, but only after verification:

1. Direct/localized builtin calls and fastcall eligibility.
2. Import/global chain optimization and environment purity.
3. `getfenv`/`setfenv`/`loadstring` as optimization blockers.
4. Table field access with known field names.
5. Stable table shapes.
6. Metatable `__index` table vs `__index` function.
7. `obj:Method()` vs `obj.Method(obj)`.
8. Type annotations for native codegen.
9. `Vector3` annotation in native code.
10. Primitive `vector` construction and reuse break-even.
11. Buffer read/write offset patterns.
12. Avoiding temporary tables/userdata/closures in hot loops.
13. Closure caching and immutable upvalues.
14. Table creation/preallocation patterns.
15. Iteration patterns.
16. Native codegen code-size budgeting.
17. Debug/profiler caveats for native code.
18. O0/O1/O2 behavior and when folding occurs.
19. Constant table/field propagation.
20. Inlining and loop-unroll cost model caveats.

---

## Final review checklist

Before handing the bundle to a user, check:

- [ ] Every document has source snapshot metadata.
- [ ] Every final rule has evidence labels.
- [ ] Every benchmark result has raw data and checksum.
- [ ] Every source claim has file/function/commit reference.
- [ ] Every unverified claim is isolated in document 08.
- [ ] No project-specific recommendation is presented as a general Luau rule.
- [ ] The final skill draft is concise enough to be reusable.
- [ ] The benchmark suite can be rerun after Luau updates.

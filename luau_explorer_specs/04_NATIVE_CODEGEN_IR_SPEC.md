# Native CodeGen, IR, and Lowering Research Spec

This spec defines the exploration for Luau native code generation, IR construction, IR optimizations, native lowering, type specialization, VM exits, and native-code size limits.

---

## Required final document

Produce:

```text
04_Native_CodeGen_IR_And_Lowering_Report.md
```

This document must separate:

- Open-source Luau CodeGen behavior.
- Roblox Studio `--!native` / `@native` behavior.
- Platform-specific behavior.
- Measured behavior.

---

## Required sections

```markdown
# Native CodeGen IR and Lowering Report

## 1. Source snapshot metadata
## 2. Executive summary
## 3. Native-codegen scope and limitations
## 4. Bytecode-to-IR pipeline
## 5. IR instruction taxonomy
## 6. IR optimization passes
## 7. Constant folding and propagation in IR
## 8. Type tags, guards, and VM exits
## 9. Number arithmetic lowering
## 10. Vector lowering
## 11. Buffer lowering
## 12. Table/global/userdata lowering
## 13. Builtin lowering
## 14. Control-flow lowering
## 15. Register pressure, block layout, and sync points
## 16. Code size limits and failure modes
## 17. Studio validation: profiler and code-size output
## 18. Practical rules
## 19. Open questions
```

---

## Source questions

Answer these from `CodeGen/` source:

1. What is the native-codegen entry point?
2. What bytecode format is consumed?
3. How are bytecode registers represented in IR?
4. How are Luau value tags represented?
5. How are type annotations fed into CodeGen?
6. Which operations generate guards?
7. Which guards exit to the VM?
8. What is an exit sync block, and when is the VM stack synchronized?
9. Which IR passes run before lowering?
10. Where is IR constant folding performed?
11. Which constants are folded at IR level rather than compiler/bytecode level?
12. How are substitutions applied?
13. What native builtins/lowerings exist for `math`, `bit32`, `buffer`, `vector`, tables, strings, and userdata?
14. Which operations are not lowered and call out to helpers?
15. Which operations allocate?
16. Which operations can deopt or fall back to interpreted execution?
17. How are block layout and fallthrough chosen?
18. How are function calls represented?
19. Which calls can be inlined or use feedback vectors?
20. What code shapes cause native lowering failures?

---

## Roblox Studio behavior questions

Verify with Studio where possible:

1. Does `--!native` compile all functions or only profitable functions?
2. Does `@native` compile only the annotated function?
3. Does top-level code execute natively?
4. How does the Script Profiler mark native functions?
5. How does `debug.dumpcodesize()` report native code memory?
6. What happens when a breakpoint is placed in a native function?
7. What happens when typed functions receive mismatched argument types?
8. What happens when discouraged code such as `getfenv` is used?
9. What are the visible code-size limits and error messages?
10. Does server/client behavior differ?

---

## Required native-codegen decision table

Fill this table.

| Source pattern | Native eligible? | IR shape | Native lowering | Guards/exits | Code-size risk | Source proof | Studio proof | Benchmark proof |
|---|---:|---|---|---|---:|---|---|---|
| Typed numeric add | ? | ? | ? | ? | ? | ? | ? | ? |
| Untyped numeric add | ? | ? | ? | ? | ? | ? | ? | ? |
| Typed Vector3 component access | ? | ? | ? | ? | ? | ? | ? | ? |
| Untyped `.X/.Y/.Z` access | ? | ? | ? | ? | ? | ? | ? | ? |
| `vector.create` | ? | ? | ? | ? | ? | ? | ? | ? |
| `vector.dot` | ? | ? | ? | ? | ? | ? | ? | ? |
| `buffer.readf32` const offset | ? | ? | ? | ? | ? | ? | ? | ? |
| `buffer.readf32` variable offset | ? | ? | ? | ? | ? | ? | ? | ? |
| `math.floor` numeric | ? | ? | ? | ? | ? | ? | ? | ? |
| indirect builtin call | ? | ? | ? | ? | ? | ? | ? | ? |
| metatable field access | ? | ? | ? | ? | ? | ? | ? | ? |
| closure allocation in loop | ? | ? | ? | ? | ? | ? | ? | ? |
| large switch/branch function | ? | ? | ? | ? | ? | ? | ? | ? |

---

## Required experiments

### 1. Type annotation specialization

Compare:

```luau
--!native
local function untyped(a, b)
    return a + b
end

local function typed(a: number, b: number): number
    return a + b
end
```

Also compare `Vector3` and `vector` annotations if available.

Required proof:

- Script Profiler native marker.
- Bytecode comparison.
- Native code size if available.
- Runtime benchmark.
- Any visible guard/fallback differences from IR dump or source inference.

### 2. Typed Vector3 component access

Compare:

```luau
--!native
local function slow(v)
    return v.X + v.Y + v.Z
end

local function fast(v: Vector3): number
    return v.X + v.Y + v.Z
end
```

This is important because official docs specifically highlight Vector3 annotation as a native-codegen performance case. Verify in Studio.

### 3. Buffer lowering

Compare:

```luau
--!native
local function sumConst(b: buffer): number
    return buffer.readf32(b, 0) + buffer.readf32(b, 4) + buffer.readf32(b, 8)
end

local function sumBase(b: buffer, base: number): number
    return buffer.readf32(b, base) + buffer.readf32(b, base + 4) + buffer.readf32(b, base + 8)
end
```

Investigate whether constant offsets change IR/native lowering.

### 4. Large-function failure modes

Create synthetic functions that stress:

- Very large straight-line code.
- Very large single block.
- Many control-flow blocks.
- Very large module with many native functions.

Record the exact failure messages and thresholds from Studio/docs/source.

---

## Native-codegen anti-pattern list

The final report must classify anti-patterns as:

| Anti-pattern | Effect | Evidence | Workaround | Caveat |
|---|---|---|---|---|
| `getfenv` / `setfenv` | deoptimizes environment / native fallback? | ? | avoid | compatibility only |
| Mismatched typed arguments | extra checks / fallback | ? | correct annotations | runtime behavior preserved |
| Debug breakpoints | disables native execution for function? | ? | remove breakpoint for profiling | debugging caveat |
| Huge functions | codegen limit | ? | split functions or use `@native` selectively | too much splitting may block inlining |
| Complex expressions causing lowering failure | native failure | ? | split/simplify | report bug if minimal repro |
| Whole-module `--!native` everywhere | startup/memory/native budget | ? | use `@native` selectively | hot code only |

---

## Required practical rules

End with rules in this format:

```markdown
### Rule: Annotate hot native-codegen function parameters when inference is ambiguous

**Status:** DOC_VERIFIED + BENCH_VERIFIED

**Why:** Native CodeGen uses inferred/annotated types to specialize code paths. Ambiguous untyped field access can produce checks or wrong assumptions.

**Prefer:**
```luau
local function f(v: Vector3): number
    return v.X + v.Y + v.Z
end
```

**Avoid in hot native code:**
```luau
local function f(v)
    return v.X + v.Y + v.Z
end
```

**Caveats:** Only use correct annotations; mismatched calls can degrade performance or fall back.
```

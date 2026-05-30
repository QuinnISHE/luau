# Benchmark, Bytecode, and Validation Harness Spec

This spec defines how to validate Luau compiler/runtime optimization findings. It is mandatory: source-reading alone is not enough for final performance rules.

---

## Required final document

Produce:

```text
06_Benchmark_Harness_And_Results.md
```

Also produce:

```text
benches/*.luau
benches/results.csv
benches/bytecode_dumps/*.txt
benches/native_code_size_dumps/*.txt
```

---

## Benchmark principles

1. Compare source-code shapes, not full applications.
2. Measure O0, O1, O2 where possible.
3. Measure native and non-native where possible.
4. Keep Roblox API calls out of inner loops unless the benchmark is explicitly about API calls.
5. Prevent dead-code elimination or result elision by accumulating into a returned value or global sink.
6. Warm up before timing.
7. Run enough iterations to exceed timer noise.
8. Report median, min, max, standard deviation, and relative speed.
9. Record environment and Luau commit/Studio version.
10. Separate open-source Luau benchmarks from Studio benchmarks.

---

## Required benchmark metadata

Every benchmark result must include:

```text
benchmark_name:
source_file:
luau_commit:
studio_version:
os:
cpu:
architecture:
native_enabled:
optimization_level:
debug_level:
iterations:
warmup_iterations:
measurement_repetitions:
timer_used:
gc_policy:
raw_results:
median:
stdev:
checksum/result_sink:
notes:
```

---

## Required CSV schema

Create `benches/results.csv` with columns:

```csv
benchmark,variant,environment,luau_commit,studio_version,os,cpu,arch,opt_level,native,strict_mode,iterations,repetitions,median_ms,min_ms,max_ms,stdev_ms,relative_to_baseline,result_checksum,notes
```

---

## Required bytecode diff harness

For each source snippet, compile/dump bytecode at all available optimization levels.

Expected artifact layout:

```text
benches/bytecode_dumps/
  fastcall_math_abs_O0.txt
  fastcall_math_abs_O1.txt
  fastcall_math_abs_O2.txt
  vector_constant_O0.txt
  vector_constant_O1.txt
  vector_constant_O2.txt
```

The benchmark report must show meaningful excerpts, not full dumps for every case.

---

## Required native validation in Roblox Studio

For Studio/native-codegen tests:

1. Use `--!native` or `@native` as needed.
2. Verify Script Profiler shows `<native>` for tested functions.
3. Run the same benchmark with native disabled or in a non-native copy.
4. Use `debug.dumpcodesize()` and save output.
5. Record whether breakpoints, type mismatch, discouraged code, or code-size limits prevented native execution.

---

## Generic benchmark runner template

Use or adapt this structure:

```luau
--!strict

local REPS = 20
local ITERS = 1_000_000
local sinkNumber = 0
local sinkVector = vector.zero

local function now(): number
    return os.clock()
end

local function runOne(name: string, f: (number) -> any)
    -- warmup
    local warm = f(math.max(10, math.floor(ITERS / 100)))
    if typeof(warm) == "number" then
        sinkNumber += warm
    elseif typeof(warm) == "vector" then
        sinkVector += warm
    end

    local times = table.create(REPS)
    local checksum = 0

    for r = 1, REPS do
        collectgarbage("collect")
        local t0 = now()
        local result = f(ITERS)
        local t1 = now()

        times[r] = (t1 - t0) * 1000

        if typeof(result) == "number" then
            checksum += result
            sinkNumber += result
        elseif typeof(result) == "vector" then
            checksum += result.x + result.y + result.z
            sinkVector += result
        end
    end

    table.sort(times)
    local median = times[math.ceil(REPS / 2)]
    print(name, "median_ms", median, "min_ms", times[1], "max_ms", times[#times], "checksum", checksum)
end
```

In Roblox Studio, replace timer/GC behavior as needed and document the exact implementation.

---

## Required benchmark families

### 1. Constant folding and bytecode shape

No runtime benchmark required if bytecode is sufficient, but include small runtime test if useful.

Cases:

- Numeric folding.
- Boolean folding.
- String concat and interpolation folding.
- Vector constant folding.
- Builtin folding.
- Constant table field propagation.
- Local-function inlining with constant args.

### 2. Fastcall call shape

Cases:

- Direct builtin call.
- Localized builtin call.
- Local table alias call.
- Dynamic table field call.
- Passed function call.
- Method call where applicable.

### 3. Type annotations and native codegen

Cases:

- Untyped numeric args vs typed numeric args.
- Untyped `Vector3` vs typed `Vector3`.
- Untyped table field access vs typed table shape if possible.
- Typed `buffer` args.
- Intentional mismatched typed calls to observe fallback/deopt behavior.

### 4. Vector/scalar break-even

Cases:

- Construct vector and use once.
- Construct vector and use 2/4/8/16/32 times.
- Manual scalar dot vs `vector.dot`.
- Manual scalar cross vs `vector.cross`.
- Vector component access/decomposition.
- Buffer-to-vector-to-buffer vs scalar buffer math.

### 5. Buffer access patterns

Cases:

- Constant offsets.
- Variable base + constant offsets.
- Dynamic offset.
- Repeated reads vs hoisted reads.
- Adjacent reads/writes.
- `readi32`, `readf32`, `readu32`, `readbits` if available.

### 6. Table shape and field access

Cases:

- Stable table shape.
- Polymorphic table shapes.
- Dynamic string key.
- Constant string key.
- Metatable `__index` table.
- Metatable `__index` function.
- Deep `__index` chain.
- Object method `:` vs `.Method(obj)`.

### 7. Allocation and closure behavior

Cases:

- Temporary table in loop.
- Hoisted/reused table.
- Closure in loop.
- Closure with no upvalues.
- Closure with immutable module upvalue.
- Closure with mutable local upvalue.
- Table literal all fields vs post-filled table.
- `table.create` vs growing table.

### 8. Loop and iteration behavior

Cases:

- Numeric `for` over buffer offsets.
- Generalized iteration over array-like table.
- `pairs`.
- `ipairs`.
- `next, t`.
- `for i = 1, #t`.

### 9. Native-code size and failure behavior

Cases:

- Many small native functions.
- One large native function.
- One function with huge straight-line block.
- One function with many branches.
- Whole-module `--!native` vs selected `@native`.

---

## Required result interpretation format

For each benchmark family:

```markdown
## Benchmark family: <name>

### Question

### Variants

### Raw results

| Variant | O0 | O1 | O2 | Native | Relative winner | Notes |
|---|---:|---:|---:|---:|---|---|

### Bytecode/native evidence

### Conclusion

### Practical rule if any

### Caveats
```

---

## Benchmark red flags

Discard or rerun a benchmark when:

- The result checksum is identical because the function never ran or was optimized away by the benchmark harness.
- One variant calls Roblox APIs and another does not.
- The benchmark body is too small relative to loop overhead.
- GC dominates and the benchmark is not about allocation.
- Native functions are not actually marked `<native>` in profiler.
- The first run differs dramatically from warmed runs and warmup was omitted.
- The timer resolution is too coarse.
- The benchmark combines multiple changes, e.g. vector + native + type annotation + table shape all at once.

---

## Required final confidence levels

Every benchmark-backed rule must include:

| Confidence | Criteria |
|---|---|
| High | Source/docs explain result and benchmark reproduces across runs. |
| Medium | Benchmark is stable, source explanation is partial. |
| Low | Benchmark observed once or source explanation uncertain. |
| Do not use | Conflicting results, noisy result, or no proof. |

---

## Required benchmark output summary

The final `06_Benchmark_Harness_And_Results.md` must end with:

```markdown
# Benchmark-backed rules summary

| Rule | Confidence | Source/docs support | Bench support | Caveats |
|---|---|---|---|---|
| TBD | TBD | TBD | TBD | TBD |
```

# Builtins, Fastcalls, Vector, and Buffer Research Spec

This spec defines the exploration for Luau builtin call optimization, fastcalls, vector support, buffer support, and related VM/native behavior.

---

## Required final document

Produce:

```text
03_Builtins_Fastcalls_Vector_Buffer_Report.md
```

This document must answer: which calls are fast because they are bytecode/core ops, which are fastcalls, which are folded, which are native-lowered, and which silently fall back to slower paths.

---

## Required sections

```markdown
# Builtins, Fastcalls, Vector, and Buffer Report

## 1. Source snapshot metadata
## 2. Executive summary
## 3. Builtin taxonomy
## 4. Fastcall dispatch model
## 5. Direct vs localized vs indirect builtin calls
## 6. Math builtin behavior
## 7. Bit32 builtin behavior
## 8. String builtin behavior
## 9. Table builtin behavior
## 10. Buffer builtin behavior
## 11. Vector type and vector library behavior
## 12. Vector vs scalar break-even benchmark plan
## 13. Roblox `Vector3` / userdata caveats
## 14. Fallback/deoptimization cases
## 15. Practical rules
## 16. Open questions
```

---

## Builtin taxonomy to produce

Create this table and fill every row with source evidence.

| Operation/call | Core bytecode op? | Fastcall? | Constant-folded? | Native lowered? | Allocates? | Fallback cases | Source proof | Benchmark proof |
|---|---:|---:|---:|---:|---:|---|---|---|
| `a + b` numbers | ? | n/a | ? | ? | no? | metamethod/type mismatch? | ? | ? |
| `a + b` vectors | ? | n/a | ? | ? | no? | type mismatch? | ? | ? |
| `math.abs(x)` | ? | ? | ? | ? | ? | non-number | ? | ? |
| `math.floor(x)` | ? | ? | ? | ? | ? | non-number | ? | ? |
| `math.max(a,b)` | ? | ? | ? | ? | ? | non-number? varargs? | ? | ? |
| `bit32.band(a,b)` | ? | ? | ? | ? | ? | non-number | ? | ? |
| `bit32.extract(n, f, w)` | ? | ? | ? | ? | ? | non-constant f/w? | ? | ? |
| `string.byte(s, i)` | ? | ? | ? | ? | ? | method call? | ? | ? |
| `s:byte(i)` | ? | ? | ? | ? | ? | method call fastcall? | ? | ? |
| `table.insert(t, v)` | ? | ? | ? | ? | maybe | table shape | ? | ? |
| `table.create(n)` | ? | ? | ? | ? | yes | dynamic n? | ? | ? |
| `buffer.readf32(b, off)` | ? | ? | ? | ? | no? | bad offset, variable offset | ? | ? |
| `buffer.writef32(b, off, v)` | ? | ? | ? | ? | no? | bad offset, variable offset | ? | ? |
| `vector.create(x,y,z)` | ? | ? | ? | ? | no heap? | non-number | ? | ? |
| `vector.dot(a,b)` | ? | ? | ? | ? | no? | non-vector | ? | ? |
| `vector.cross(a,b)` | ? | ? | ? | ? | no? | non-vector | ? | ? |
| `vector.normalize(v)` | ? | ? | ? | ? | no? | zero vector? | ? | ? |

---

## Fastcall model questions

Answer these from source:

1. Where are builtin IDs defined?
2. Where is call-shape recognition performed?
3. Which source calls produce FASTCALL bytecode?
4. Which source calls are recognized as imports?
5. Which calls are recognized when localized, e.g. `local abs = math.abs`?
6. Which calls are not recognized when hidden behind tables/functions?
7. Which fastcalls are partial and fall back for unsupported argument types?
8. How are argument counts represented?
9. How are return counts represented?
10. Which fastcalls have special constant-argument variants?

---

## Direct/localized/indirect call benchmark matrix

For each builtin category, benchmark:

```luau
local function direct(x)
    return math.abs(x)
end

local abs = math.abs
local function localized(x)
    return abs(x)
end

local M = math
local function viaLocalTable(x)
    return M.abs(x)
end

local function passedFunction(f, x)
    return f(x)
end

local function tableFunction(t, x)
    return t.abs(x)
end
```

Required outputs:

- Bytecode at O0/O1/O2.
- Whether FASTCALL opcodes appear.
- Runtime benchmark.
- Native-codegen benchmark if applicable.
- Conclusion for direct/localized/indirect calls.

---

## Vector research requirements

### Source questions

1. How is the primitive `vector` represented in the VM?
2. What is the tested build's vector width?
3. How are vector constants represented in bytecode?
4. Which vector arithmetic operators are implemented as core operations?
5. Which vector library functions are fastcalls?
6. Which vector library functions are constant-folded?
7. Which vector functions have native lowering?
8. Does `vector.create` allocate heap memory or produce an immediate/tagged value?
9. How does vector component access lower?
10. Is vector component write unsupported because vectors are immutable?
11. What happens when vector operations receive non-vector arguments?
12. Do vector operations use float32 or double inputs/results at each step?
13. Are vector constants embedded into bytecode at O1/O2?
14. Does native CodeGen use SIMD instructions for vector arithmetic on x64/arm64?

### Required vector benchmark families

#### 1. Construction cost

```luau
local function construct3(n: number)
    local acc = vector.zero
    for i = 1, n do
        acc += vector.create(i, i + 1, i + 2)
    end
    return acc
end
```

Compare to scalar triplets:

```luau
local function scalar3(n: number)
    local ax, ay, az = 0, 0, 0
    for i = 1, n do
        ax += i
        ay += i + 1
        az += i + 2
    end
    return ax, ay, az
end
```

#### 2. Reuse break-even

Generate benchmarks where each constructed vector is used for 1, 2, 4, 8, 16, and 32 arithmetic operations before decomposition.

Output:

| Operations per constructed vector | Scalar time | Vector time | Winner | Notes |
|---:|---:|---:|---|---|
| 1 | ? | ? | ? | ? |
| 2 | ? | ? | ? | ? |
| 4 | ? | ? | ? | ? |
| 8 | ? | ? | ? | ? |
| 16 | ? | ? | ? | ? |
| 32 | ? | ? | ? | ? |

#### 3. Dot/cross/manual comparison

Compare:

```luau
local function dotScalar(ax, ay, az, bx, by, bz)
    return ax*bx + ay*by + az*bz
end

local function dotVector(a: vector, b: vector)
    return vector.dot(a, b)
end
```

Also compare manual vector arithmetic:

```luau
local function dotVectorManual(a: vector, b: vector)
    local c = a * b
    return c.x + c.y + c.z
end
```

#### 4. Component access and decomposition

Benchmark vector math that ends with:

```luau
return v.x, v.y, v.z
```

Compare to scalar math that never constructs vectors.

#### 5. Buffer-to-vector and vector-to-buffer

Benchmark:

- `buffer.readf32` x3 -> scalar ops -> `buffer.writef32` x3.
- `buffer.readf32` x3 -> `vector.create` -> vector ops -> split/write.
- If vector buffer APIs exist in the tested runtime, benchmark those too.

---

## Buffer research requirements

Answer:

1. Which buffer read/write functions are builtins or fastcalls?
2. Which have native lowering?
3. Does a constant byte offset produce better bytecode/native code than a variable offset?
4. Are adjacent reads combined or kept separate?
5. Are bounds checks hoisted, eliminated, or repeated?
6. Do writes invalidate native assumptions or store caches?
7. Which buffer APIs are added in recent releases, e.g. integer or bit read/write APIs?
8. What offset/alignment patterns are fastest?
9. Does `buffer.len` fold or fastcall when buffer is known? Usually no — verify.
10. Are `buffer.copy` or `buffer.fill` optimized specially?

### Buffer benchmark matrix

| Case | O2 bytecode | Native? | Time | Notes |
|---|---|---:|---:|---|
| `readf32(b, constOff)` | ? | ? | ? | ? |
| `readf32(b, base + const)` | ? | ? | ? | ? |
| `readf32(b, i * stride + const)` | ? | ? | ? | ? |
| 3 adjacent reads | ? | ? | ? | ? |
| 3 reads reused once | ? | ? | ? | ? |
| 3 reads reused many times | ? | ? | ? | ? |
| write constant offset | ? | ? | ? | ? |
| write variable offset | ? | ? | ? | ? |

---

## Required practical rules

The report must end with a list like:

```markdown
## Practical rules

### Prefer obvious builtin calls for fastcall eligibility
Status: DOC_VERIFIED + SOURCE_VERIFIED + BENCH_VERIFIED
...

### Use vectors when construction is amortized
Status: BENCH_VERIFIED
...

### Avoid decomposing vectors immediately after one operation
Status: BENCH_VERIFIED or HYPOTHESIS
...

### Prefer constant buffer offsets where possible
Status: SOURCE_VERIFIED + BENCH_VERIFIED or HYPOTHESIS
...
```

Do not finalize a vector/scalar break-even rule without measurements.

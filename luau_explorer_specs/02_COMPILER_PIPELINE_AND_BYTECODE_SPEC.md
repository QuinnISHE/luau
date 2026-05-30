# Compiler Pipeline, Bytecode, and Constant-Folding Research Spec

This document defines the required investigation for Luau's compiler pipeline, bytecode generation, constant folding, import resolution, inlining, and optimization levels.

---

## Required final document

Produce:

```text
01_Luau_Compiler_Pipeline_Map.md
02_Bytecode_Optimization_And_Folding_Matrix.md
```

Both documents must be source-grounded and benchmark/bytecode-validated.

---

## Document 01: `Luau_Compiler_Pipeline_Map.md`

### Required sections

```markdown
# Luau Compiler Pipeline Map

## 1. Source snapshot metadata
## 2. Executive summary
## 3. Source-to-bytecode pipeline diagram
## 4. Parse / AST stage
## 5. Constant tracking and folding stage
## 6. Builtin recognition and import chain handling
## 7. Table shape and constant table propagation
## 8. Local function inlining
## 9. Loop unrolling
## 10. Bytecode emission
## 11. Peephole or post-emission optimizations
## 12. Debug information and optimization interaction
## 13. Optimization-level matrix
## 14. Open questions
```

### Required pipeline diagram

Include an ASCII diagram similar to this, but based on the source:

```text
Luau source
  -> lexer/parser
  -> AST
  -> builtin/global import discovery
  -> constant/value tracking
  -> constant folding
  -> cost model / inline decisions
  -> bytecode generation
  -> bytecode builder / constant table
  -> bytecode proto + debug/type info
  -> VM interpreter or native CodeGen input
```

If the real pipeline differs, correct the diagram.

---

## Optimization-level matrix

Fill this table by source inspection and bytecode tests:

| Optimization | O0 | O1 | O2 | Source file/function | Test snippet | Bytecode proof | Notes |
|---|---:|---:|---:|---|---|---|---|
| Numeric constant folding | ? | ? | ? | TBD | TBD | TBD | TBD |
| Boolean folding | ? | ? | ? | TBD | TBD | TBD | TBD |
| `and`/`or` folding | ? | ? | ? | TBD | TBD | TBD | TBD |
| Unary minus folding | ? | ? | ? | TBD | TBD | TBD | TBD |
| String length folding | ? | ? | ? | TBD | TBD | TBD | TBD |
| String concatenation folding | ? | ? | ? | TBD | TBD | TBD | TBD |
| String interpolation folding | ? | ? | ? | TBD | TBD | TBD | TBD |
| Vector constant folding | ? | ? | ? | TBD | TBD | TBD | TBD |
| Foldable builtin calls | ? | ? | ? | TBD | TBD | TBD | TBD |
| `math.pi` / `math.huge` constants | ? | ? | ? | TBD | TBD | TBD | TBD |
| Import chain resolution | ? | ? | ? | TBD | TBD | TBD | TBD |
| Table field constant propagation | ? | ? | ? | TBD | TBD | TBD | TBD |
| Local function inlining | ? | ? | ? | TBD | TBD | TBD | TBD |
| Loop unrolling | ? | ? | ? | TBD | TBD | TBD | TBD |
| Function return-count optimization | ? | ? | ? | TBD | TBD | TBD | TBD |
| Builtin argument/return-count assumptions | ? | ? | ? | TBD | TBD | TBD | TBD |

---

## Document 02: `Bytecode_Optimization_And_Folding_Matrix.md`

This document must be a detailed “what folds?” manual.

### Required sections

```markdown
# Bytecode Optimization and Folding Matrix

## 1. Source snapshot metadata
## 2. Reading guide
## 3. Constants and literal folding
## 4. Numeric expression folding
## 5. Boolean/control-flow folding
## 6. String folding
## 7. Vector folding
## 8. Builtin folding
## 9. Table-field constant propagation
## 10. Import/global chain optimization
## 11. Inlining and folding interaction
## 12. Bytecode before/after gallery
## 13. Non-folded examples and why they do not fold
## 14. Practical rules
## 15. Open questions
```

---

## Required folding categories

Investigate and document each of these.

### 1. Literal constants

Test:

```luau
local a = 1 + 2
local b = 3 * 4
local c = 10 / 5
local d = 10 // 3
local e = 2 ^ 8
local f = 10 % 4
```

Record:

- Folded value.
- Bytecode at O0/O1/O2.
- Whether integer-related bytecode changes exist.
- Whether floating-point edge cases such as `0/0`, `1/0`, `math.huge`, `NaN`, and signed zero are preserved.

### 2. Boolean and comparison folding

Test:

```luau
local a = not false
local b = 1 < 2
local c = 1 == 1
local d = true and 123
local e = false or 456
local f = nil or "fallback"
```

Record whether boolean/control-flow folding merely substitutes constants or removes branches.

### 3. String folding

Test:

```luau
local a = "a" .. "b"
local b = `hello {"world"}`
local c = #"abcdef"
local d = string.char(65)
local e = string.sub("abcdef", 2, 4)
```

Required investigation points:

- Source limit for folded string size.
- Which string functions fold.
- Whether folding changes at O1 vs O2.
- Whether interpolation folds when only some fragments are constant.
- Whether methods and functions differ, e.g. `string.sub(s, ...)` vs `s:sub(...)`.

### 4. Vector folding

Test:

```luau
local a = vector.create(1, 2, 3)
local b = vector.create(1, 2, 3) + vector.create(4, 5, 6)
local c = vector.create(1, 2, 3) * 2
local d = vector.dot(vector.create(1, 2, 3), vector.create(4, 5, 6))
local e = vector.cross(vector.create(1, 0, 0), vector.create(0, 1, 0))
```

Required investigation points:

- Whether `vector.create` becomes a bytecode vector constant.
- Which vector arithmetic folds.
- Which vector library functions fold.
- Whether vector constants use 3-wide or 4-wide storage in the tested Luau build.
- Whether `vector` globals must be direct calls to fold.
- Whether component access of constant vectors folds.

### 5. Builtin folding

Create a matrix of builtin calls:

| Builtin | Constant-folded? | Fastcalled? | Native lowered? | Argument constraints | Source file/function | Test snippet |
|---|---:|---:|---:|---|---|---|
| `math.abs` | ? | ? | ? | ? | ? | ? |
| `math.floor` | ? | ? | ? | ? | ? | ? |
| `math.max` | ? | ? | ? | ? | ? | ? |
| `math.min` | ? | ? | ? | ? | ? | ? |
| `math.clamp` | ? | ? | ? | ? | ? | ? |
| `math.sqrt` | ? | ? | ? | ? | ? | ? |
| `bit32.band` | ? | ? | ? | ? | ? | ? |
| `bit32.bor` | ? | ? | ? | ? | ? | ? |
| `bit32.extract` | ? | ? | ? | constant field/width? | ? | ? |
| `string.byte` | ? | ? | ? | ? | ? | ? |
| `string.sub` | ? | ? | ? | ? | ? | ? |
| `table.create` | ? | ? | ? | ? | ? | ? |
| `buffer.read*` | ? | ? | ? | const offset? | ? | ? |
| `buffer.write*` | ? | ? | ? | const offset? | ? | ? |
| `vector.dot` | ? | ? | ? | vector args | ? | ? |
| `vector.cross` | ? | ? | ? | vector args | ? | ? |

Do not assume all `math` calls fold. Verify per function.

### 6. Import/global chains

Test:

```luau
local function direct(x)
    return math.max(x, 1)
end

local max = math.max
local function localized(x)
    return max(x, 1)
end

local M = math
local function viaTable(x)
    return M.max(x, 1)
end

local function indirect(f, x)
    return f(x, 1)
end
```

Record:

- Bytecode differences.
- Fastcall eligibility.
- Whether import-chain optimization is preserved.
- Whether `getfenv`, `setfenv`, or `loadstring` deoptimizes the environment.

### 7. Constant table propagation

Test:

```luau
local config = { a = 2, b = 4 }
local function foo(x)
    return x * config.a
end
```

Then test mutations:

```luau
local config = { a = 2, b = 4 }
config.a = 10
local function foo(x)
    return x * config.a
end
```

Required investigation:

- What counts as “mutated”?
- Is indirect mutation detected?
- Are nested tables folded?
- Are table fields folded across local functions?
- Which optimization level enables this?

### 8. Inlining and folding interaction

Test:

```luau
local SCALE = 2
local function mul(x)
    return x * SCALE
end

local function use(x)
    return mul(x) + mul(3)
end
```

Required investigation:

- Is `mul(3)` folded after inlining?
- Are constants propagated into function arguments?
- What cost model controls this?
- How do constant arguments change the cost model?
- What breaks inlining: varargs, recursion, closures, upvalues, complex control flow, large functions, multiple returns?

---

## Required bytecode gallery format

For every important transformation:

```markdown
### Example: <name>

#### Source

```luau
-- source here
```

#### O0 bytecode

```text
<bytecode dump>
```

#### O1 bytecode

```text
<bytecode dump>
```

#### O2 bytecode

```text
<bytecode dump>
```

#### Interpretation

- What changed:
- Why it changed:
- Source file/function responsible:
- Caveats:
```

---

## Required practical-rule format

At the end, produce rules like:

```markdown
### Rule: Keep foldable constants visibly constant

**Prefer**

```luau
local INV_DT = 60
local MAX_STEP = 1 / INV_DT
```

**Avoid in hot paths**

```luau
local settings = getSettings()
local MAX_STEP = 1 / settings.invDt
```

**Why:** The compiler can only fold expressions whose inputs are proven constant. Dynamic table/function inputs block folding.

**Evidence:** SOURCE_VERIFIED + BENCH_VERIFIED.
```

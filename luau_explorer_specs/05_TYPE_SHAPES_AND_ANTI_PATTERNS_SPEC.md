# Type Annotations, Table Shapes, Code Shapes, and Anti-Patterns Research Spec

This spec defines the practical source-code-shape research. It turns low-level compiler/VM facts into general Luau performance rules.

---

## Required final document

Produce:

```text
05_Type_Annotations_Table_Shapes_And_AntiPatterns.md
```

This document must be a practical guide, but every rule must be tied back to source/docs/benchmarks.

---

## Required sections

```markdown
# Type Annotations, Table Shapes, and Anti-Patterns

## 1. Source snapshot metadata
## 2. Executive summary
## 3. Type annotations and type inference
## 4. Number/math-heavy code shapes
## 5. Vector and Vector3 code shapes
## 6. Table field access and table shapes
## 7. Metatables and object patterns
## 8. Globals, imports, locals, and environment purity
## 9. Functions, inlining, closures, and upvalues
## 10. Loop shapes and iteration
## 11. Allocation patterns and GC pressure
## 12. Buffer access code shapes
## 13. Builtin call shapes
## 14. Debug/profiling caveats
## 15. Do / avoid / benchmark table
## 16. Open questions
```

---

## Required source-code pattern matrix

Fill this table.

| Pattern | Preferred? | Why | Evidence | Benchmark | Caveat |
|---|---:|---|---|---|---|
| `obj.field` with known field | yes | inline cache / known field | ? | ? | uniform shape matters |
| `obj[dynamicKey]` | no for hot field access | dynamic lookup | ? | ? | needed for maps |
| `obj["field"]` constant string | maybe same as dot | compiler can optimize? | ? | ? | verify |
| metatable `__index` table | acceptable | method lookup optimized | ? | ? | keep direct table |
| metatable `__index` function | avoid hot | blocks fast path? | ? | ? | sometimes needed |
| deep `__index` chain | avoid | extra lookup | ? | ? | cold code ok |
| `obj:Method()` | prefer | namecall/method fast path | ? | ? | not same as builtin fastcall |
| `obj.Method(obj)` | avoid | misses method syntax optimization | ? | ? | maybe local function case |
| direct `math.max` | prefer | import/fastcall | ? | ? | no dynamic env |
| `local max = math.max` | acceptable | localized fastcall | ? | ? | verify bytecode |
| indirect builtin wrapper | avoid hot | not obvious to compiler | ? | ? | can inline sometimes |
| `getfenv`/`setfenv` | avoid | impure environment | ? | ? | compatibility only |
| table literal with all fields | prefer | table template/shape | ? | ? | cold code less important |
| `{}` then sequential fields | acceptable? | capacity prediction? | ? | ? | verify |
| `table.create(n)` | prefer for known array size | prealloc | ? | ? | `n` must be reasonable |
| closure inside loop | avoid if allocates | allocation/GC | ? | ? | closure caching cases |
| immutable upvalues | prefer | cheaper capture/caching | ? | ? | value may still mutate |
| `for ... in t` | prefer for table iteration? | optimized generalized iteration | ? | ? | ordering semantics |
| `for i=1,#t` | sometimes slower for table arrays | extra table reads | ? | ? | numeric loops over buffers differ |
| repeated buffer read | avoid if reused | repeated load/check | ? | ? | register pressure tradeoff |
| hoisted buffer read | prefer if reused | fewer reads | ? | ? | avoid hoisting unused reads |
| huge native function | avoid | code size/block limits | ? | ? | split carefully |

---

## Required type annotation investigation

Answer:

1. Which type annotations affect bytecode generation?
2. Which type annotations affect native codegen?
3. Does `--!strict` itself affect generated code, or only the available type information? Verify.
4. How do annotations on arguments differ from local variable annotations?
5. How do casts `::` affect typechecking and codegen?
6. Which annotations are especially useful for `Vector3`, `vector`, `buffer`, tables, and function returns?
7. Can incorrect annotations make native code slower due to checks/fallbacks?
8. What is the difference between Luau primitive `vector` and Roblox `Vector3` from a codegen standpoint?

---

## Required table-shape investigation

Answer:

1. What is a table shape in Luau source/VM terminology?
2. How does known field access get optimized?
3. What breaks the field access inline cache?
4. How many different table shapes can a function tolerate before performance drops? Benchmark.
5. Are object-like table literals optimized differently from post-filled tables?
6. Does `table.freeze` affect optimization or only correctness? Verify.
7. Does `const` affect optimization or only binding immutability? Verify.
8. Does mutation of a table prevent constant table propagation?
9. Does indirect mutation prevent propagation?
10. Are nested constant tables propagated?

---

## Required metatable/object investigation

Benchmark these object patterns:

```luau
-- Direct fields
local obj = { x = 1, y = 2 }
local function direct(o)
    return o.x + o.y
end

-- __index table
local Class = {}
Class.__index = Class
function Class:getX()
    return self.x
end
local obj2 = setmetatable({ x = 1 }, Class)

-- __index function
local obj3 = setmetatable({ x = 1 }, {
    __index = function(t, k)
        return 0
    end,
})
```

Measure:

- Field read.
- Method call via `:`.
- Method call via `.Method(obj)`.
- Method cached into local variable.
- Object shapes stable vs varied.

---

## Required closure/upvalue investigation

Answer:

1. What are immutable upvalues in Luau implementation terms?
2. When are upvalues captured by value?
3. When are extra upvalue objects allocated?
4. When does closure caching apply?
5. What breaks closure caching?
6. What happens when function objects are used as table keys?
7. What is the cost of closures inside hot loops?

Benchmark:

```luau
local function noClosure(n)
    local acc = 0
    for i = 1, n do
        acc += i
    end
    return acc
end

local function closureInLoop(n)
    local acc = 0
    for i = 1, n do
        local f = function()
            return i
        end
        acc += f()
    end
    return acc
end

local IMM = 5
local function closureMaybeCached()
    return function(x)
        return x + IMM
    end
end
```

---

## Required allocation/GC investigation

Catalog operations that allocate:

| Operation | Allocates? | Allocation type | Avoidance strategy | Source proof | Benchmark proof |
|---|---:|---|---|---|---|
| table literal | yes | table | reuse/preallocate | ? | ? |
| function expression | maybe | closure | cache/hoist | ? | ? |
| string concat | yes? maybe folded | string | fold/pre-size/buffer | ? | ? |
| vector create | heap? value? | TBD | benchmark | ? | ? |
| Vector3 create | userdata/value? Roblox-specific | TBD | benchmark | ? | ? |
| buffer.create | yes | buffer | allocate outside hot loop | ? | ? |
| table.insert | no new table, maybe resize | array storage | table.create | ? | ? |

Do not claim `vector.create` heap-allocates unless source proves it. The VM represents primitive vectors specially; verify.

---

## Required anti-pattern format

For each anti-pattern:

```markdown
### Anti-pattern: <name>

**Avoid in:** hot loops / native functions / allocation-sensitive code / all code.

**Bad:**
```luau
...
```

**Better:**
```luau
...
```

**Why:**

**Evidence:**
- SOURCE_VERIFIED:
- DOC_VERIFIED:
- BENCH_VERIFIED:

**Caveats:**

**When it is acceptable:**
```

---

## Minimum anti-patterns to cover

- `getfenv`, `setfenv`, and `loadstring` in optimized code.
- Dynamic builtin dispatch when fastcall is wanted.
- Method call shape mistakes: `obj.Method(obj)` vs `obj:Method()`.
- `__index` functions and deep metatable chains in hot paths.
- Varying table shapes in a hot function.
- Allocating temporary tables in tight loops.
- Creating closures in tight loops.
- Using large whole-module native compilation instead of targeted `@native` where code-size budget matters.
- Untyped `Vector3`/userdata arguments in native hot code.
- Constructing vectors for a single operation then immediately decomposing them, if benchmarks prove scalar wins.
- Rereading the same buffer fields repeatedly when values are reused, if benchmarks prove hoisting wins.
- Unverified “Lua optimization folklore” that is false or weaker in Luau.

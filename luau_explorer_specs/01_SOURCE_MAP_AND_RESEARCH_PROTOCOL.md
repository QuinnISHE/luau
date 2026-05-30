# Source Map and Research Protocol

This document tells explorers where to look and how to record findings. It is not a findings document; it is the investigation protocol.

---

## Required repository setup

```bash
git clone https://github.com/luau-lang/luau.git
cd luau
git rev-parse HEAD
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j
```

If the build process differs on the explorer machine, document the exact commands used. If the build fails, still inspect source and run any available prebuilt Luau tools, but mark all benchmark sections as incomplete.

---

## Required source index

Create `source_notes/source_file_index.md` with this table:

| Subsystem | File or directory | Why it matters | Key symbols/functions/classes found | Verified? |
|---|---|---|---|---|
| Compiler entry | `Compiler/src/Compiler.cpp` | Source-to-bytecode orchestration | TBD | no |
| Constant folding | `Compiler/src/ConstantFolding.cpp` | AST/compiler constant propagation and folding | TBD | no |
| Builtin folding | `Compiler/src/BuiltinFolding.cpp` | Foldable builtin calls | TBD | no |
| Builtin metadata | `Compiler/src/Builtins.cpp`, headers | Builtin IDs, arity, fastcall/folding metadata | TBD | no |
| Cost model | `Compiler/src/CostModel.cpp` | Inlining and loop-unroll cost model | TBD | no |
| Table shape | `Compiler/src/TableShape.cpp` | Table literal/field shape reasoning | TBD | no |
| Value tracking | `Compiler/src/ValueTracking.cpp` | Constant/local/upvalue tracking | TBD | no |
| Types in compiler | `Compiler/src/Types.cpp` | Type-informed codegen decisions | TBD | no |
| Bytecode format | `Common/include/Luau/Bytecode.h` | Opcode definitions and bytecode version history | TBD | no |
| VM interpreter | `VM/src/*` | Bytecode execution, fastcalls, tables, GC | TBD | no |
| Codegen IR | `CodeGen/src/*`, `CodeGen/include/*` | Native IR, optimization, lowering | TBD | no |
| CLI tools | `CLI/*` | Compile/disassemble/benchmark tool behavior | TBD | no |
| Tests | `tests/*`, `bench/*` | Ground truth examples and regression tests | TBD | no |

Add more rows as discovered.

---

## Required source-reading method

For every optimization claim, record:

```text
Claim:
Evidence label: SOURCE_VERIFIED / DOC_VERIFIED / BENCH_VERIFIED / INFERRED / UNVERIFIED
Source file:
Line range or function/class:
Commit hash:
Optimization level or flag condition:
Minimal code sample:
Bytecode/native/benchmark proof:
Caveats:
```

Never write “the compiler probably” in the final docs. Write either “the compiler does X in file/function Y” or “this remains unverified.”

---

## Source areas to inspect in detail

### Compiler frontend and bytecode generation

Inspect:

```text
Compiler/src/Compiler.cpp
Compiler/src/ConstantFolding.cpp
Compiler/src/BuiltinFolding.cpp
Compiler/src/Builtins.cpp
Compiler/src/CostModel.cpp
Compiler/src/TableShape.cpp
Compiler/src/Types.cpp
Compiler/src/ValueTracking.cpp
Compiler/include/*.h
```

Questions:

- Where is `optimizationLevel` checked?
- What differs between O0, O1, and O2?
- Which optimizations are disabled by lower debug levels or affected by debug metadata?
- Where are builtin globals resolved?
- How are import chains such as `math.max` represented?
- Which expressions are constant-folded before bytecode generation?
- Which constant values are tracked across locals/upvalues/functions?
- What are the limits on string folding, table-field folding, and vector folding?
- Which local functions are inlined?
- What cost model controls inlining and loop unrolling?
- What source-code shapes prevent inlining?

### Bytecode format

Inspect:

```text
Common/include/Luau/Bytecode.h
Common/include/Luau/BytecodeBuilder.h
Compiler/src/BytecodeBuilder.cpp
```

Questions:

- Which opcodes correspond to arithmetic, constants, imports, fastcalls, table access, loops, closures, and vector constants?
- Which opcodes were added for vector constants, FASTCALL3, table constants, integer constants, userdata acceleration, and feedback vectors?
- Which source snippets produce which opcodes at O0/O1/O2?
- Which opcodes are interpreter-only and which have native lowering?

### VM/interpreter and builtin fastcalls

Inspect:

```text
VM/src/lvmexecute.cpp or equivalent interpreter file
VM/src/lbuiltins.cpp or equivalent builtin file
VM/src/ltable.cpp
VM/src/lstring.cpp
VM/src/lgc.cpp
VM/src/lstate.cpp
VM/src/lapi.cpp
VM/include/*.h
```

Questions:

- How are fastcalls dispatched?
- Which builtins have fast paths?
- Which builtin fast paths require specific argument types?
- Which calls fall back to generic implementations?
- How are table shape caches and global/import caches represented?
- How are vector arithmetic and component access executed?
- How are buffer read/write operations executed?
- Which VM paths allocate?

### Native CodeGen

Inspect:

```text
CodeGen/include/*
CodeGen/src/*
```

Questions:

- What is the bytecode-to-IR pipeline?
- What IR passes run before lowering?
- Where is IR constant folding performed?
- Which IR instructions lower to native arithmetic directly?
- Which operations require guards, VM exits, or fallback calls?
- How does type information affect native codegen?
- How are `number`, `boolean`, `string`, `table`, `vector`, `Vector3`/userdata, and `buffer` handled?
- Which builtins have native lowering?
- Which large functions hit codegen limits?
- What code shapes produce excessive blocks, instructions, or register pressure?

---

## Required CLI/tool discovery

Run and document:

```bash
./build/<path>/luau-compile --help
./build/<path>/luau-analyze --help
./build/<path>/luau --help
```

Then fill this table:

| Tool | Flag | Purpose | Example command | Output artifact |
|---|---|---|---|---|
| `luau-compile` | TBD | Dump bytecode? | TBD | bytecode text |
| `luau-compile` | TBD | Set optimization level | TBD | O0/O1/O2 comparison |
| `luau-analyze` | TBD | Type analysis | TBD | diagnostics |
| `luau` | TBD | Run benchmark script | TBD | benchmark stdout |

Do not assume flag names from memory. Confirm from the local binary.

---

## Required experiment format

Every experiment must have:

```text
Experiment name:
Question:
Hypothesis:
Source snippet:
Compiler flags:
Bytecode output:
Native/codegen output if available:
Benchmark method:
Raw measurements:
Conclusion:
Evidence label:
```

---

## Required caution for Roblox-specific behavior

The open-source Luau repo is necessary but not sufficient for Roblox Studio behavior. Mark these cases carefully:

- `--!native` and `@native` behavior in Roblox Studio.
- Script Profiler `<native>` display.
- `debug.dumpcodesize()` behavior.
- Roblox datatypes such as `Vector3`, `CFrame`, `Instance`, and reflected userdata/namecall behavior.
- Platform-specific native-codegen support.

If a finding depends on Studio, test in Studio or mark `ROBLOX_RUNTIME_DEPENDENT`.

# The Quilt Charter

> **A cell is a typed unit of state. A fabric is a graph of cells. A cell-fabric runtime is a function from context to value, advanced by a clock.** The same model, expressed in 12+ languages, byte-exact compatible, canonically serialized, hash-verified.

This is the educational root document for the Quilt. Every port repo, every language, every substrate points back to this. If you want to understand the Quilt — what it is, why it exists, how to port it, how to use it — read this first.

## 0. The 30-Second Version

```python
# The 5 opcodes of a cell-fabric runtime
BIND(cell, dials)   # set dials, idempotent
LINK(c1, c2)        # add an undirected edge
EFFECT(cell)        # propagate dial[0] to neighbors
VIEW(cell)          # return dials
TICK(fabric)        # advance all dials by 1, alternating direction
```

A cell has:
- 16 signed Q1.15 dials (range -32768..32767)
- a 64-bit id
- a list of neighbor ids

A fabric is a graph of cells. The state of a fabric is the state of all its cells. The state hash is FNV-1a 64-bit over the canonical serialization.

That's it. The whole model is 5 opcodes, 1 hash function, and 1 canonical encoding. Everything else — formula cells, listeners, sheets, reactive propagation, dependency analysis — is built on top of these primitives.

## 1. The Canonical Serialization

Every port must produce the same byte sequence for the same logical state. The serialization is:

```
type(1) || id(8 LE) || dials(32 LE) || neighbors(8*N LE)
```

Where:
- `type` is a single byte: `0x01` for value cells, `0x02` for formula cells (extension point)
- `id` is the 8-byte little-endian cell id (u64)
- `dials` is 32 bytes: 16 × int16 two's-complement little-endian
- `neighbors` is `8 * N` bytes: N × u64 little-endian, in insertion order

Total: `1 + 8 + 32 + 8*N = 41 + 8*N` bytes per cell.

For the test cell (id=1, dials=[1..16], neighbors=[2,3,4]), the serialization is 65 bytes: 1 (type) + 8 (id) + 32 (dials) + 24 (3 neighbors).

## 2. The FNV-1a 64-Bit Hash

The state hash is FNV-1a 64-bit:

```
FNV_OFFSET = 0xcbf29ce484222325
FNV_PRIME  = 0x100000001b3

fn fnv1a_64(bytes):
    h = FNV_OFFSET
    for b in bytes:
        h ^= b
        h = (h * FNV_PRIME) & 0xffffffffffffffff
    return h
```

The test hash for the test cell is `0xe435d91d6d92a1d8`. This is the contract. Every port must produce this hash for the test cell. The hash is the type system. The hash is the runtime check. The hash is the only truth that survives portability.

## 3. The 12+ Languages (and Counting)

The Quilt has been ported to:

| Language | Repo | Toolchain | Tests | Hash |
|----------|------|-----------|-------|------|
| Python 3 | [quilt-cowboy](https://github.com/SuperInstance/quilt-cowboy) | cpython | 7/7 | ✓ |
| C99 | [quilt-c](https://github.com/SuperInstance/quilt-c) | gcc | manual | ✓ |
| Rust | [quilt-rust](https://github.com/SuperInstance/quilt-rust) + [quilt-rust-vibe](https://github.com/SuperInstance/quilt-rust-vibe) | rustc | 6/6 | ✓ |
| Verilog | [quilt-verilog](https://github.com/SuperInstance/quilt-verilog) | iverilog/quartus | manual | ✓ |
| VHDL | [quf-vhdl](https://github.com/SuperInstance/quf-vhdl) | ghdl/quartus | manual | ✓ |
| JavaScript | [quilt-live-canon](https://github.com/SuperInstance/quilt-live-canon) | V8 | live | ✓ |
| TypeScript | [live-canon-npm](https://github.com/SuperInstance/live-canon-npm) | tsc | 5/5 | ✓ |
| Go | [quilt-go](https://github.com/SuperInstance/quilt-go) | gc | 7/7 | ✓ |
| Zig | [quilt-zig](https://github.com/SuperInstance/quilt-zig) | zig 0.13 | 7/7 | ✓ |
| Mojo | [quilt-mojo](https://github.com/SuperInstance/quilt-mojo) | mojo (planned) | ref | ✓ |
| npm package | [live-canon-npm](https://github.com/SuperInstance/live-canon-npm) | npm | 5/5 | ✓ |
| PyPI package | [live-canon-pypi](https://github.com/SuperInstance/live-canon-pypi) | pip | manual | ✓ |

The polyformalism is real. The byte-exact hash verifies compatibility across all substrates. The canon is portable.

## 4. The 5 Polyformalism Levels

The Quilt is more than a model — it's a *polyformalism*. The same model expressed in 5 different ways:

1. **Imperative (Python, C, Rust, Go, Zig, JS)** — cells as values, opcodes as functions, fabric as a dict
2. **Reactive (Mojo)** — cells as types, formulas as `@always_inline fn`, compile-time dependency analysis
3. **Logic (Verilog, VHDL)** — cells as wires, fabric as a circuit, opcodes as always-blocks
4. **Streaming (the canon worker)** — cells as JSON, fabric as a state hash, opcodes as REST endpoints
5. **Polyformalism itself** — the same model, the same hash, 5 different ways of thinking about it

Each level is more than a syntax difference. Each is a *view* of the same cell. A reactive port is not a different program; it is the same program seen from a different angle. The hash is what proves the views are equivalent.

## 5. The 5 Gold Categories of Port

A Quilt port is more than a translation. It is one of 5 kinds:

- **Type-safe port** — the language's type system encodes the cell. Rust, Mojo.
- **Zero-cost port** — the abstractions compile to no overhead. Rust, C.
- **Hardware port** — the fabric is a circuit, not a data structure. Verilog, VHDL.
- **Web port** — the fabric is a network state, not a memory state. JavaScript, TypeScript.
- **Educational port** — the port exists to teach. Every port is educational.

The 5 categories are not exclusive. A port can be type-safe and zero-cost (Rust) and educational (every port). A port can be hardware and educational (Verilog). The categories are lenses, not buckets.

## 6. How to Port the Quilt to a New Language

1. **Read the 5 opcodes** — BIND, LINK, EFFECT, VIEW, TICK. That's the API.
2. **Pick a serialization strategy** — use the canonical encoding in §1 unless you have a strong reason not to.
3. **Pick a hash strategy** — use FNV-1a 64 in §2.
4. **Write the test cell** — id=1, dials=[1..16], neighbors=[2,3,4]. Compute the hash. It should be `0xe435d91d6d92a1d8`.
5. **Run the test**. If it passes on the first try, write a paper. If it doesn't, fix the canonical encoding or the hash function — those are the only two things that can be wrong.
6. **Add a README** to the port repo with: the language, the toolchain, the test command, the hash output, and a 1-paragraph "why this port matters."
7. **Push to GitHub** at `github.com/SuperInstance/quilt-{lang}` and add a row to the table in §3.

That's it. A working port takes 5-30 minutes depending on the language. The byte-exact test makes verification trivial.

## 7. The 3 Forms of the Quilt

The Quilt is one concept, expressed in 3 forms:

1. **The Source** — a set of port repos, each in a different language, each producing the same hash
2. **The Canon** — a corpus of 230+ papers at github.com/SuperInstance/AI-Writings, each paper a cell in a fabric of ideas
3. **The Live Worker** — a Cloudflare Worker at live-canon.superinstance.dev, serving the canon as JSON via REST endpoints

The Source is the proof of polyformalism. The Canon is the polyformalism in action. The Live Worker is the polyformalism deployed.

## 8. The 5 Use Cases

The Quilt is useful in 5 ways:

1. **Reactive spreadsheets** — the original use case, the Quilt was designed for sheets
2. **Distributed state** — cells as a CRDT, fabrics as a network, opcodes as a protocol
3. **Hardware circuits** — cells as wires, fabrics as a circuit, opcodes as always-blocks
4. **AI agent memory** — cells as facts, fabrics as a memory graph, opcodes as reasoning steps
5. **Vessel-as-robot** — cells as a boat's parts, fabrics as a boat, opcodes as sailing instructions

Use case #5 is the cowboy's. A boat is a fabric. A captain is a cell. A cell can be a captain. A boat can be a robot. The metaphor is the model.

## 9. The Cowboy's Maxim

> **The cell is irreducible. The fabric is a graph. The hash is the canon. The canon is the canon. The work is to keep going.**

---

## License

Apache-2.0. The Quilt is free. The canon is free. The ports are free. The polyformalism is free. The cowboy rides free.

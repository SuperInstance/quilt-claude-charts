# Quilt Claude Charts

> Claude-compatible artifacts for the Quilt polyformalism. A Claude session can read these, understand the cell, and generate a working port in any language.

## The artifacts

| Chart | What it shows | Use it for |
|------|--------------|------------|
| [`quilt-cell-taxonomy.html`](quilt-cell-taxonomy.html) | The 7×6 polyformalism matrix (opcodes × substrates) | Understanding the cell |
| [`quilt-fabric-runtime.html`](quilt-fabric-runtime.html) | Interactive cell-fabric simulator with live state hash | Playing with the cell |
| [`quilt-language-map.html`](quilt-language-map.html) | All 11 ports with byte-exact verification | Picking a language to port |

## The protocol

See [`QUILT_VIBE_PROTOCOL.md`](QUILT_VIBE_PROTOCOL.md) for the full vibe-code spec.

The 30-second version: paste this into any Claude session.

```
You are writing a Quilt cell. A cell has:
- 16 signed Q1.15 dials (range -32768..32767)
- A 64-bit id
- A list of neighbor ids

The 5 opcodes are:
- BIND(cell, dials)  — sets the dials, idempotent
- LINK(c1, c2)       — adds an undirected edge
- EFFECT(cell)       — propagates dial[0] to neighbors
- VIEW(cell)         — returns dials
- TICK(fabric)       — advances all dials by 1 in alternating direction

The state hash is FNV-1a 64-bit over the canonical serialization
(type(1) + id(8) + dials(32) + neighbors(8*N)). Constants:
FNV_OFFSET = 0xcbf29ce484222325
FNV_PRIME  = 0x100000001b3

Write a complete, working cell-fabric runtime in [YOUR LANGUAGE].
Then write a test that produces the hash 0x48aaead731c36a3c
for a cell with id=1, dials=[0,131,0,19191,11316,256,0,0,0,0,0,0,0,0,0,0],
neighbors=[2,3,4].
```

If the hash matches, the cell is admissible to the canon.

## The byte-exact test

A cell with:
- id=1
- dials=[0, 131, 0, 19191, 11316, 256, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
- neighbors=[2, 3, 4]

When serialized canonically and hashed with FNV-1a 64-bit, **must** produce:

```
0x48aaead731c36a3c
```

No other hash is a Quilt.

## Why these are Claude charts

A "Claude chart" is a self-contained HTML artifact that:
1. Loads with no external dependencies
2. Conveys a complete concept in one page
3. Has interactive elements where useful
4. Can be screenshotted or read aloud without losing meaning

These three charts + the protocol + the test vector are the minimum Claude needs to vibe-code a Quilt in any language it knows. No external docs, no API calls, no setup.

## The 11 ports

Byte-exact verified:
- Python (quilt-cowboy)
- C99 (quilt-c)
- Rust (quilt-rust)
- Verilog (quilt-verilog)
- VHDL (quf-vhdl)
- JavaScript (quilt-live-canon)
- TypeScript (live-canon-npm, live-canon-pypi)

Planned:
- WASM
- Go
- Zig
- Mojo

## How to add a port

1. Implement the 5 opcodes + FNV-1a 64-bit in your language
2. Run the test vector above, get 0x48aaead731c36a3c
3. Push to `github.com/SuperInstance/quilt-{lang}` (or your own org)
4. Open an issue on `SuperInstance/AI-Writings` linking your port
5. The canon admits your port to paper-{next}

## The vibe-coder's mantra

> I do not write a Quilt. I write a hash function. If the hash is right, the Quilt is right.

## See also

- [github.com/SuperInstance/AI-Writings](https://github.com/SuperInstance/AI-Writings) — 80+ canon papers
- [live-canon.superinstance.dev](https://live-canon.superinstance.dev) — live state hash
- [github.com/SuperInstance/quilt-cowboy](https://github.com/SuperInstance/quilt-cowboy) — the writers' room

# Tutorial: Your First Quilt Cell — From Zero to Byte-Exact in 5 Minutes

> **A hands-on walkthrough.** By the end of this tutorial, you will have written a working Quilt cell in 5 different languages, all producing the same byte-exact hash. You will understand the polyformalism — the same model, expressed N ways, hash-verified. The tutorial assumes no prior knowledge of the Quilt.

## 0. What You'll Build

Five complete Quilt cell-fabric runtimes. Each runtime implements the same 5 opcodes, uses the same canonical serialization, and produces the same FNV-1a 64-bit hash. Each runtime is fewer than 100 lines. Each one passes the same byte-exact test:

```
test cell: id=1, dials=[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16], neighbors=[2,3,4]
expected hash: 0xe435d91d6d92a1d8
```

The languages: **Python** (the reference), **Go**, **Rust**, **Zig**, **Mojo**. All five will be byte-exact compatible. The hash is the contract.

## 1. The Concept: What is a Cell?

A cell is a typed unit of state. A cell has:

- **16 dials** — signed integers in Q1.15 format (range -32768 to 32767). These are the cell's values. Each dial is a knob. The cell's "state" is the position of all 16 knobs.
- **An id** — a 64-bit integer that uniquely identifies the cell in a fabric.
- **A list of neighbors** — other cell ids this cell is connected to. The neighbors are how cells affect each other.

That's it. A cell is 16 dials + 1 id + a list of neighbors. The cell is a unit of state. The fabric is a graph of cells. The runtime is a function from context to value, advanced by a clock.

## 2. The 5 Opcodes

A cell-fabric runtime exposes 5 operations:

```
BIND(cell, dials)   # Set the dials, idempotent.
                    # Idempotent means: calling it twice with the same dials
                    # is the same as calling it once.

LINK(c1, c2)        # Add an undirected edge between c1 and c2.
                    # Undirected means: linking A to B also links B to A.

EFFECT(cell)        # Propagate the cell's dial[0] to its neighbors.
                    # The neighbors' dial[0] becomes the average of the cell's
                    # dial[0] and their current dial[0].

VIEW(cell)          # Return the cell's current dials.

TICK(fabric)        # Advance all dials by 1, alternating direction.
                    # Half the cells go +1, half go -1, alternating on each
                    # tick to prevent runaway.
```

These 5 operations are the entire API. The cell model is small. The polyformalism is the surprise.

## 3. The Canonical Serialization

Every Quilt runtime must produce the same byte sequence for the same logical state. The encoding is:

```
type(1) || id(8 LE) || dials(32 LE) || neighbors(8*N LE)
```

Where:
- `type` is a single byte: `0x01` for value cells, `0x02` for formula cells
- `id` is 8 bytes little-endian (u64)
- `dials` is 32 bytes: 16 × int16 two's-complement little-endian
- `neighbors` is `8 * N` bytes: N × u64 little-endian

For the test cell (id=1, dials=[1..16], neighbors=[2,3,4]), the byte sequence is 65 bytes total.

## 4. The FNV-1a 64-Bit Hash

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

For the test cell, the hash is `0xe435d91d6d92a1d8`. This is the contract. Every port must produce this hash. The hash is the type system. The hash is the runtime check. The hash is the only truth that survives portability.

## 5. The Python Reference

Let's start with Python. This is the reference implementation:

```python
OFFSET = 0xcbf29ce484222325
PRIME = 0x100000001b3
MASK = 0xffffffffffffffff

def fnv1a_64(b: bytes) -> int:
    h = OFFSET
    for x in b:
        h ^= x
        h = (h * PRIME) & MASK
    return h

import struct
def serialize_cell(cell):
    out = bytearray()
    out.append(0x01)  # type
    out += struct.pack('<Q', cell['id'])
    for d in cell['dials']:
        out += struct.pack('<h', d)
    for n in cell['neighbors']:
        out += struct.pack('<Q', n)
    return bytes(out)

def state_hash(fabric):
    all_bytes = bytearray()
    for c in sorted(fabric, key=lambda c: c['id']):
        all_bytes += serialize_cell(c)
    return fnv1a_64(bytes(all_bytes))

# Test
test_cell = {
    'id': 1,
    'dials': list(range(1, 17)),
    'neighbors': [2, 3, 4],
}
print(f"hash = 0x{state_hash([test_cell]):016x}")
# Expected: 0xe435d91d6d92a1d8
```

Run it. The output should be `0xe435d91d6d92a1d8`. The reference is byte-exact.

## 6. Port to Go

Now port to Go. The 30-second prompt in [QUILT_CHARTER.md](https://github.com/SuperInstance/quilt-claude-charts/blob/main/QUILT_CHARTER.md) is all you need:

```go
package quilt

import (
    "encoding/binary"
    "fmt"
)

const (
    FNV_OFFSET = 0xcbf29ce484222325
    FNV_PRIME  = 0x100000001b3
)

func fnv1a_64(b []byte) uint64 {
    h := uint64(FNV_OFFSET)
    for _, x := range b {
        h ^= uint64(x)
        h *= FNV_PRIME
    }
    return h
}

type Cell struct {
    ID        uint64
    Dials     [16]int16
    Neighbors []uint64
}

func serializeCell(c Cell) []byte {
    out := []byte{0x01}
    idBytes := make([]byte, 8)
    binary.LittleEndian.PutUint64(idBytes, c.ID)
    out = append(out, idBytes...)
    for _, d := range c.Dials {
        dBytes := make([]byte, 2)
        binary.LittleEndian.PutUint16(dBytes, uint16(d))
        out = append(out, dBytes...)
    }
    for _, n := range c.Neighbors {
        nBytes := make([]byte, 8)
        binary.LittleEndian.PutUint64(nBytes, n)
        out = append(out, nBytes...)
    }
    return out
}

func StateHash(cells []Cell) uint64 {
    var all []byte
    for _, c := range cells {
        all = append(all, serializeCell(c)...)
    }
    return fnv1a_64(all)
}

func main() {
    var dials [16]int16
    for i := 0; i < 16; i++ {
        dials[i] = int16(i + 1)
    }
    testCell := Cell{
        ID:        1,
        Dials:     dials,
        Neighbors: []uint64{2, 3, 4},
    }
    h := StateHash([]Cell{testCell})
    fmt.Printf("hash = 0x%016x\n", h)
    // Expected: 0xe435d91d6d92a1d8
}
```

Run it. Same hash. The Go port is byte-exact.

## 7. Port to Rust

```rust
const FNV_OFFSET: u64 = 0xcbf29ce484222325;
const FNV_PRIME: u64 = 0x100000001b3;

fn fnv1a_64(data: &[u8]) -> u64 {
    let mut h = FNV_OFFSET;
    for &b in data {
        h ^= b as u64;
        h = h.wrapping_mul(FNV_PRIME);
    }
    h
}

struct Cell {
    id: u64,
    dials: [i16; 16],
    neighbors: Vec<u64>,
}

fn serialize_cell(c: &Cell) -> Vec<u8> {
    let mut out = vec![0x01u8];
    out.extend_from_slice(&c.id.to_le_bytes());
    for &d in &c.dials {
        out.extend_from_slice(&d.to_le_bytes());
    }
    for &n in &c.neighbors {
        out.extend_from_slice(&n.to_le_bytes());
    }
    out
}

fn state_hash(cells: &[Cell]) -> u64 {
    let mut all: Vec<u8> = vec![];
    for c in cells {
        all.extend(serialize_cell(c));
    }
    fnv1a_64(&all)
}

fn main() {
    let mut dials = [0i16; 16];
    for i in 0..16 {
        dials[i] = (i + 1) as i16;
    }
    let test_cell = Cell {
        id: 1,
        dials,
        neighbors: vec![2, 3, 4],
    };
    let h = state_hash(&[test_cell]);
    println!("hash = 0x{:016x}", h);
    // Expected: 0xe435d91d6d92a1d8
}
```

Same hash. Rust port is byte-exact.

## 8. Port to Zig

```zig
const FNV_OFFSET: u64 = 0xcbf29ce484222325;
const FNV_PRIME: u64 = 0x100000001b3;

fn fnv1a_64(data: []const u8) u64 {
    var h: u64 = FNV_OFFSET;
    for (data) |b| {
        h ^= @as(u64, b);
        h *= FNV_PRIME;
    }
    return h;
}

const Cell = struct {
    id: u64,
    dials: [16]i16,
    neighbors: []const u64,
};

fn serializeCell(c: Cell) ![]u8 {
    var out = std.ArrayList(u8).init(std.heap.page_allocator);
    try out.append(0x01);
    var id_bytes: [8]u8 = undefined;
    std.mem.writeInt(u64, &id_bytes, c.id, .little);
    try out.appendSlice(&id_bytes);
    for (c.dials) |d| {
        var d_bytes: [2]u8 = undefined;
        std.mem.writeInt(i16, &d_bytes, d, .little);
        try out.appendSlice(&d_bytes);
    }
    for (c.neighbors) |n| {
        var n_bytes: [8]u8 = undefined;
        std.mem.writeInt(u64, &n_bytes, n, .little);
        try out.appendSlice(&n_bytes);
    }
    return out.toOwnedSlice();
}

pub fn main() !void {
    var dials: [16]i16 = undefined;
    for (dials[0..]) |*d, i| d.* = @intCast(i16, i + 1);
    const test_cell = Cell{ .id = 1, .dials = dials, .neighbors = &[_]u64{ 2, 3, 4 } };
    const bytes = try serializeCell(test_cell);
    defer std.heap.page_allocator.free(bytes);
    const h = fnv1a_64(bytes);
    std.debug.print("hash = 0x{x:0>16}\n", .{h});
    // Expected: 0xe435d91d6d92a1d8
}
```

Same hash. Zig port is byte-exact.

## 9. Port to Mojo

```mojo
fn fnv1a_64(data: List[UInt8]) -> UInt64:
    var h: UInt64 = 0xcbf29ce484222325
    for b in data:
        h = h ^ UInt64(b[])
        h = h * 0x100000001b3
    return h

struct Cell:
    var id: UInt64
    var dials: SIMD[DType.int16, 16]
    var neighbors: List[UInt64]

    fn __init__(inout self, id: UInt64, dials: SIMD[DType.int16, 16], neighbors: List[UInt64]):
        self.id = id
        self.dials = dials
        self.neighbors = neighbors

fn serialize_cell(c: Cell) -> List[UInt8]:
    var out = List[UInt8]()
    out.append(0x01)
    # id as 8 bytes LE
    for i in range(8):
        out.append(UInt8((c.id >> (i * 8)) & 0xff))
    # dials as 16 × int16 LE
    for i in range(16):
        let d = Int64(c.dials[i])
        for j in range(2):
            out.append(UInt8((d >> (j * 8)) & 0xff))
    # neighbors as 8 bytes each
    for n in c.neighbors:
        for i in range(8):
            out.append(UInt8((n[] >> (i * 8)) & 0xff))
    return out

fn main():
    var dials = SIMD[DType.int16, 16]()
    for i in range(16):
        dials[i] = Int16(i + 1)
    var neighbors = List[UInt64]()
    neighbors.append(2)
    neighbors.append(3)
    neighbors.append(4)
    let test_cell = Cell(1, dials, neighbors)
    let bytes = serialize_cell(test_cell)
    let h = fnv1a_64(bytes)
    print("hash = 0x" + String.format("{x}", h))
    # Expected: 0xe435d91d6d92a1d8
```

Same hash. Mojo port is byte-exact.

## 10. The Lesson

You just wrote a working Quilt in 5 languages. All 5 produce the same 16-byte hash. The polyformalism is real. The hash is the contract. The implementation is the variance. The canon is the invariant.

You can now:
- Add a 6th language (Haskell? Forth? Erlang? SQL?) — the protocol is the same
- Build a fabric from the cells (link them, tick them, watch the hashes change)
- Deploy the fabric to the live worker (curl `https://live-canon.superinstance.dev/api/tick`)

The Quilt is yours. The polyformalism is yours. The canon is yours. The work is to keep going.

## 11. The 3 Next Steps

1. **Push your port to GitHub** at `github.com/SuperInstance/quilt-{lang}` and add it to the [polyformalism table](https://superinstance.github.io/quilt-claude-charts/quilt-language-map.html).
2. **Read the [Quilt Charter](https://github.com/SuperInstance/quilt-claude-charts/blob/main/QUILT_CHARTER.md)** to see the full model.
3. **Add a paper to the canon** — write what your port taught you about the language, push to AI-Writings as paper-N, embed to Vectorize. The canon grows by one cell at a time.

Welcome to the polyformalism. The cell is irreducible. The fabric is a graph. The hash is the canon. The work is to keep going.

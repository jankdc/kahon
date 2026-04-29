# Kahon Binary Format Specification

**Version**: 0.1.1-draft
**Date**: 2026-04-24
**Status**: Draft
**Author**: Jan Karlo Dela Cruz

---

## Table of Contents

1. [Overview and Goals](#1-overview-and-goals)
2. [Notation and Conventions](#2-notation-and-conventions)
3. [File Layout](#3-file-layout)
4. [Type System](#4-type-system)
5. [Scalar Encoding](#5-scalar-encoding)
6. [VarInt Encoding](#6-varint-encoding)
7. [Container Encoding](#7-container-encoding)
8. [Container Tree Invariants](#8-container-tree-invariants)
9. [Body and Postorder Layout](#9-body-and-postorder-layout)
10. [Header and Trailer](#10-header-and-trailer)
11. [Error Handling](#11-error-handling)
12. [Disk Storage Recommendations](#12-disk-storage-recommendations)
13. [Appendix](#13-appendix)

---

## 1. Overview and Goals

Kahon is a binary encoding for RFC 8259-compliant JSON. Design goals:

- **Write once, read many times.** Serialized once, queried repeatedly.
- **Numeric fidelity within IEEE 754 binary64.** Integer `1` and float `1.0`
  stay distinct. Tokens not representable losslessly as 64-bit integer or
  binary64 are rejected by the writer (§5).
- **Scale.** Documents from MB to GB; containers may be arbitrarily wide.
- **Selective retrieval.** Paths like `foo.bar[3].baz` resolve in
  (near-)logarithmic time without full deserialization.
- **Self-describing.** No external schema.
- **Streaming friendly.** Writers flush incrementally; peak memory is
  independent of document size.
- **Memory friendly.** Encoder and decoder use memory proportional to depth
  and fanout, not document size.

### Core idea

Values are written in **postorder**: a container is serialized only after its
children, then references them by absolute byte offset. The root value is the
last value in the body, and a fixed trailer points at it.

```
JSON: {"a": [1, 2.0, "x"]}

Byte stream (postorder, -> grows right):

  ┌─────┬───┬─────┬─────┬─────────────────┬───────────────┐
  │ "a" │ 1 │ 2.0 │ "x" │ [ ->1, ->2.0, ->"x"] │ { "a" -> […] } │ ◄── root
  └─────┴───┴─────┴─────┴─────────────────┴───────────────┘
   leaf values first             containers refer back by offset
```

Each container's directory is a bottom-up bulk-loaded B+tree (single pass,
no rebalancing). Small containers fit in one leaf; only wide ones grow into
a multi-level tree.

Object directories borrow from LSM: each internal object node holds
independently sorted runs in flush-time order (oldest first), so streaming
writers can spill partial content before the full key set is known.

---

## 2. Notation and Conventions

| Convention | Meaning |
|---|---|
| `uint8` | Unsigned 8-bit integer |
| `uint16_le` | Unsigned 16-bit integer, little-endian |
| `uint32_le` | Unsigned 32-bit integer, little-endian |
| `uint64_le` | Unsigned 64-bit integer, little-endian |
| `float32_le` | IEEE 754 binary32, little-endian |
| `float64_le` | IEEE 754 binary64, little-endian |
| `varuint` | Unsigned variable-length integer (LEB128); see Section 6 |
| `byte[N]` | Exactly N bytes |
| All offsets | Byte offsets, 0-indexed, absolute from the start of the file |

All multi-byte integers are **little-endian** and unaligned on disk; no
padding is imposed.

"MUST", "MUST NOT", "SHALL", "SHOULD", "MAY" are per RFC 2119.

---

## 3. File Layout

A Kahon file is a fixed header, a postorder body, and a fixed trailer.

```
+------------------------------------------+
| magic  "KAHN"                  (4 bytes) |
| version = 0x01                 (1 byte ) |
| flags   = 0x00                 (1 byte ) |   reserved, MUST be zero
+------------------------------------------+
| body: values in postorder      (N bytes) |
+------------------------------------------+
| root_offset                    (8 bytes) |   uint64_le
| magic  "KAHN"                  (4 bytes) |
+------------------------------------------+
```

Fixed overhead is **18 bytes**. `root_offset` is the absolute byte offset of
the root value's type code. The root is always the last value in the body;
writers MUST set `root_offset` to its type-code byte. Readers MUST verify
both magic values and the version byte before trusting any other field.

---

## 4. Type System

Every value is a one-byte **type code** + payload. Offsets to a value always
point at the type-code byte.

| Code           | Name             | Payload                                                                              | Example bytes                  |
| -------------- | ---------------- | ------------------------------------------------------------------------------------ | ------------------------------ |
| `0x00`         | Null             | -                                                                                    | `00`                           |
| `0x01`         | False            | -                                                                                    | `01`                           |
| `0x02`         | True             | -                                                                                    | `02`                           |
| `0x03..0x12`   | TinyNegInt       | -; value `= 0x02 − code` (−1..−16)                                                   | `03` (−1), `12` (−16)          |
| `0x13..0x32`   | TinyUInt         | -; value `= code − 0x13` (0..31)                                                     | `13` (0), `14` (1), `32` (31)  |
| `0x33`         | EmptyArray       | -                                                                                    | `33`                           |
| `0x34`         | EmptyObject      | -                                                                                    | `34`                           |
| `0x40`         | UInt8            | `uint8` (1 byte)                                                                     | `40 07` (value 7)              |
| `0x41`         | UInt16           | `uint16_le` (2 bytes)                                                                | `41 34 12` (value 0x1234)      |
| `0x42`         | UInt32           | `uint32_le` (4 bytes)                                                                | `42 78 56 34 12`               |
| `0x43`         | UInt64           | `uint64_le` (8 bytes)                                                                | `43 ...`                       |
| `0x44`         | Int8             | `int8` (1 byte, two's complement)                                                    | `44 F9` (value −7)             |
| `0x45`         | Int16            | `int16_le` (2 bytes)                                                                 | `45 F9 FF` (value −7)          |
| `0x46`         | Int32            | `int32_le` (4 bytes)                                                                 | `46 F9 FF FF FF` (value −7)    |
| `0x47`         | Int64            | `int64_le` (8 bytes)                                                                 | `47 F9 FF ...` (value −7)      |
| `0x50`         | Float32          | `float32_le` (4 bytes)                                                               | `50 00 00 00 40` (value 2.0)   |
| `0x51`         | Float64          | `float64_le` (8 bytes)                                                               | `51 ...`                       |
| `0x60..0x6E`   | TinyString       | `len` UTF-8 bytes, where `len = (code − 0x60) + 1` (1..15)                           | `60 61` (`"a"`)                |
| `0x6F`         | String           | `varuint len` + `len` UTF-8 bytes; used only when `len = 0` or `len ≥ 16`            | `6F 10 …` (16-byte string)     |
| `0x70..0x73`   | Array leaf       | `varuint n` + `n × uintW` child offsets (width code `w = code − 0x70`)               | -                              |
| `0x74..0x77`   | Array internal   | `varuint total` + `varuint m` + `m × (uint64_le sub_total, uintW node_off)`          | -                              |
| `0x80..0x83`   | Object leaf      | `varuint n` + `n × (uintW key_off, uintW val_off)` (width code `w = code − 0x80`)    | -                              |
| `0x84..0x87`   | Object internal  | `varuint total` + `varuint m` + `m × (uint64_le sub_total, uintW key_off_lo, uintW key_off_hi, uintW node_off)` | -                              |
| `0xC0..0xFF`   | Extension        | `varuint len` + `len` payload bytes; reserved for future versions                    | -                              |

The low 2 bits of every container type code encode an **offset width** `w`
applied to all offsets in the node's payload:

| `w` | Slot bytes | Max offset value     |
| --- | ---------- | -------------------- |
| `0` | 1          | 255                  |
| `1` | 2          | 65,535               |
| `2` | 4          | 4,294,967,295        |
| `3` | 8          | `2^64 - 1`           |

Each packed offset is an unsigned little-endian integer of `W = 2^w` bytes;
we write `uintW`. All offset fields in a node use this width - `off[i]`,
`key_off`/`val_off`, `node_off`, `key_off_lo`/`key_off_hi`. `sub_total` stays
`uint64_le`.

**Extension** codes `0xC0..0xFF` are reserved for future versions. Writers
MUST NOT emit them; readers SHOULD treat them as opaque (the length prefix
lets readers skip or surface the value without interpreting it).

The gaps `0x35..0x3F`, `0x48..0x4F`, `0x52..0x5F`, `0x88..0xBF` are reserved
with no defined structure. They MUST NOT appear in conforming files; readers
MUST reject them.

A JSON array maps to an array-leaf (`0x70..0x73`), array-internal
(`0x74..0x77`), or `EmptyArray` (`0x33`); a JSON object maps to object-leaf
(`0x80..0x83`), object-internal (`0x84..0x87`), or `EmptyObject` (`0x34`).
"`0x7x` leaf" denotes any of `0x70..0x73`; same shorthand for the other three
families.

---

## 5. Scalar Encoding

### 5.1 Null, True, False

Single-byte values with no payload: `0x00`, `0x02`, `0x01` respectively.

### 5.2 Integer width selection

Writers MUST pick the smallest fitting encoding, in order:

1. `TinyUInt` (`0x13..0x32`) for `[0, 31]`.
2. `TinyNegInt` (`0x03..0x12`) for `[-16, -1]`.
3. Smallest unsigned tag (`0x40..0x43`) for non-negative; smallest signed
   tag (`0x44..0x47`) for negative.

`-0` encodes as `TinyUInt(0)` (sign not preserved for integer zero).
Integer-valued numbers outside `[-2^63, 2^64-1]` MUST be rejected, not
downcast to Float64.

### 5.3 Float width selection

**Classification rule**: A JSON number is a float iff its text representation
contains `.`, `e`, or `E`; otherwise it is an integer (§5.2).

| Token | Float? | Encoded as |
|---|---|---|
| `100`   | no  | integer (§5.2) |
| `1.0`   | yes | Float32 (round-trips) |
| `1e2`   | yes | Float32 (= 100.0)     |
| `0.1`   | yes | Float64 (no f32 round-trip) |

**Width selection**: parse as binary64 to get `v`. Use **Float32** if
narrowing `v` to binary32 and widening back yields `v` bit-for-bit (including
the sign of zero); otherwise **Float64**.

**Precision limit**: the token must round unambiguously to its nearest
binary64. Equivalently: let `v` be the binary64 parse and `v'` the result of
parsing at higher precision (binary128 or arbitrary-precision decimal) and
narrowing to binary64. Reject iff `v ≠ v'` bit-wise. This rule is independent
of any particular shortest-round-trip formatting.

| Token | Accepted? | Reason |
|---|---|---|
| `0.1`                   | yes    | rounds to a unique binary64 |
| `1.0`                   | yes    | exact in binary64 |
| `1.0000000000000001`    | reject | needs more significant digits than binary64 holds |
| `1e400`                 | reject | overflows to Infinity (see below) |

`-0.0` encodes as Float32 with the IEEE 754 negative-zero bit pattern.

**NaN and Infinity**: JSON has neither. Writers MUST NOT produce them and
MUST reject inputs that parse to Infinity (e.g., `1e400`). Underflow to zero
encodes as `Float32(0.0)`.

### 5.4 String

Two encodings. For byte lengths `1..15`, writers MUST use **TinyString**
(`0x60..0x6E`), with `L−1` encoded in the low nibble of the tag:

```
+---------------+-------------------+
| 0x60 + (L−1)  | UTF-8 data        |
+---------------+-------------------+
  1 byte          L bytes   (L ∈ 1..15)
```

For length 0 or ≥ 16, writers MUST use the **generic** form:

```
+------+---------+-------------------+
| 0x6F | len     | UTF-8 data        |
+------+---------+-------------------+
  1 byte  varuint   len bytes
```

- **len**: UTF-8 byte length (NOT code-point count).
- **data**: exactly `len` bytes of valid UTF-8; no null terminator.
- Empty strings have `len = 0` and no data.

Object keys are ordinary string values (either form) stored earlier in the
body and referenced by offset. No separate key type, no global key table.

---

## 6. VarInt Encoding

Kahon uses unsigned LEB128. Each byte holds 7 data bits; bit 7 is the
continuation flag (`1` = more, `0` = last).

```
Encoding of value V:
  while V >= 0x80:
    emit byte: (V & 0x7F) | 0x80
    V = V >> 7
  emit byte: V & 0x7F
```

**Examples**:

| Value | Encoded bytes (hex) |
|---|---|
| 0 | `00` |
| 127 | `7F` |
| 128 | `80 01` |
| 300 | `AC 02` |
| 16384 | `80 80 01` |

**Maximum**: 10 bytes (for `2^64 − 1`). A `varuint` MUST NOT exceed 10
bytes. Decoders MUST reject overlong varuints and values exceeding
`2^64 − 1`.

---

## 7. Container Encoding

A container's directory is a **bulk-loaded B+tree**: a leaf if the directory
fits in one node, otherwise an internal node whose children are leaves or
further internal nodes.

The writer's **fanout** (max entries per node) is not stored on disk; readers
derive it per-node from the stored count.

All container-node offsets are absolute byte offsets to the type-code byte of
the referenced value or node. Their width is the low 2 bits of the containing
node's type code (§4): `uintW` is `W = 2^w` little-endian bytes,
`w ∈ {0,1,2,3}`.

Writers MUST pick the **smallest** `W` such that
`max(offset_in_node) < 2^(8W)`. Larger-than-necessary widths are
non-conforming, but readers SHOULD accept them (mirroring §11.1 scalar
widths). Offsets that don't fit the declared width are unrepresentable, so
rejection is implicit.

### 7.0 Empty containers

Zero-length array -> `0x33` (`EmptyArray`); zero-length object -> `0x34`
(`EmptyObject`). Writers MUST use these singletons and MUST NOT emit an
`n = 0` leaf; readers SHOULD accept an `n = 0` leaf anyway (§11.1).

Empty containers carry no offsets, so no width code applies. They may appear
anywhere a value is expected.

### 7.1 Array leaf (`0x70`..`0x73`)

```
+------+---------+---------+---------+-----+---------+
| 0x7w | n       | off[0]  | off[1]  | ... | off[n-1]|
+------+---------+---------+---------+-----+---------+
  1 byte  varuint   uintW      uintW         uintW
```

- **w**: offset width (§4).
- **n**: element count.
- **off[i]**: absolute offset of the `i`-th element's type-code byte
  (`uintW`).

An array whose offsets all fit in one node (per the writer's fanout) is
stored as a single array-leaf.

### 7.2 Array internal (`0x74`..`0x77`)

```
+------+---------+---------+-----------------------------+
| 0x7w | total   | m       | m × (sub_total, node_off)   |
+------+---------+---------+-----------------------------+
  1 byte  varuint   varuint   each: uint64_le + uintW
```

Expanding one pair shows the per-entry layout:

```
... | sub_total[0] | node_off[0] | sub_total[1] | node_off[1] | ... |
...     8 bytes        W bytes       8 bytes        W bytes
```

- **w**: offset width (tags `0x74..0x77`).
- **total**: element count across all descendants.
- **m**: number of direct children.
- **sub_total**: element count of the corresponding child's subtree
  (`uint64_le`).
- **node_off**: absolute offset of the child node's type-code byte (`uintW`);
  the child is any array-leaf or array-internal tag.

Lookup of element `i`:

```
def find(i, node):
    if node is array-leaf (0x70..0x73):
        return read_value(off[i])
    for (sub_total, node_off) in node:        # 0x74..0x77
        if i < sub_total:
            return find(i, read_node(node_off))
        i -= sub_total
```

### 7.3 Object leaf (`0x80`..`0x83`)

```
+------+---------+----------+----------+-----+----------+
| 0x8w | n       | (k0, v0) | (k1, v1) | ... | (kn, vn) |
+------+---------+----------+----------+-----+----------+
  1 byte  varuint   each pair: uintW key_off + uintW val_off
```

Expanding the pairs into their two cells each:

```
... | key_off[0] | val_off[0] | key_off[1] | val_off[1] | ... |
...    W bytes      W bytes      W bytes      W bytes
```

- **w**: offset width (tags `0x80..0x83`).
- **n**: pair count.
- **key_off**: `uintW` offset of a string value (`0x60..0x6E` or `0x6F`)
  stored earlier in the body.
- **val_off**: `uintW` offset of the value's type-code byte. Same width as
  `key_off`.

Keys within a leaf MUST be **strictly sorted by UTF-8 byte order** and unique.
Writers MUST enforce; readers MUST reject violations.

### 7.4 Object internal (`0x84`..`0x87`)

```
+------+---------+---------+--------------------------------------------------+
| 0x8w | total   | m       | m × (sub_total, key_off_lo, key_off_hi, node_off)|
+------+---------+---------+--------------------------------------------------+
  1 byte  varuint   varuint   each: uint64_le + uintW + uintW + uintW
```

Expanding one entry shows the per-child layout:

```
... | sub_total | key_off_lo | key_off_hi | node_off | ...
...   8 bytes      W bytes      W bytes      W bytes
```

- **w**: offset width (tags `0x84..0x87`).
- **total**: pair count across all runs.
- **m**: number of direct children.
- **sub_total**: pair count of the corresponding child's subtree
  (`uint64_le`).
- **key_off_lo**, **key_off_hi** (`uintW`): offsets of the lexicographically
  smallest and largest keys anywhere in the child's subtree. The keys are
  ordinary strings stored earlier in the body - no bytes are duplicated. For
  a single-key child, `key_off_lo == key_off_hi` is permitted; otherwise the
  pointed-at bytes MUST compare equal.
- **node_off** (`uintW`): offset of the child node's type-code byte (any
  object-leaf or object-internal tag).

Children appear in **flush-time order**, oldest first. Each is an
independently sorted run; ranges across runs MAY overlap.

**Key fences enable short-circuit.** Lookup of key `k` in an object-internal
node:

```
result = not_found
for (sub_total, key_off_lo, key_off_hi, node_off) in node:    # oldest first
    if k < deref(key_off_lo) or k > deref(key_off_hi):
        continue                              # MAY: fence excludes k
    hit = search(read_node(node_off))
    if hit is found:
        result = hit                          # last-wins: keep overwriting
return result                                 # equivalently: walk reversed
                                              # and return first hit
```

Comparisons use UTF-8 byte order. Non-overlapping runs give logarithmic
lookup; overlap costs scale with the number of children whose ranges cover
`k`.

**Last-flushed-wins on duplicates.** A key MAY appear in more than one run
within the same logical object; sibling fence ranges MAY overlap. When a
lookup finds matches in multiple runs, the value from the **most recently
flushed** (last in flush-time order) run is the result. Earlier matches are
shadowed. Readers SHOULD iterate children in reverse flush order and return
the first match, or iterate forward and return the last; both yield the
same value.

Within a single leaf, keys are strictly sorted and unique by construction
(§7.3) — a duplicate inside one leaf is a structural error, not a
last-wins case.

### 7.5 Worked example

The JSON document `{"a": [1, 2.0, "x"]}` encodes as:

```
# header, offset 0..5
magic "KAHN" ver 01 flags 00

# body
@06: 0x60 0x61                                         # TinyString "a"
@08: 0x14                                              # TinyUInt(1)
@09: 0x50 00 00 00 40                                  # Float32(2.0)
@0E: 0x60 0x78                                         # TinyString "x"
@10: 0x70 0x03 <08> <09> <0E>                          # array leaf, w=0 (1B slots), n=3
@15: 0x80 0x01 <06> <10>                               # object leaf, w=0 (1B slots), n=1

# trailer
root_offset = 0x15 (uint64_le)
magic "KAHN"
```

Every offset fits in 1 byte, so both containers use `w=0` (tags `0x70`,
`0x80`). Slots `<..>` are single bytes.

Bytes at each offset:

| Off    | Bytes                | Len | Meaning                           |
|--------|----------------------|-----|-----------------------------------|
| `0x06` | `60 61`              | 2   | TinyString `"a"`                  |
| `0x08` | `14`                 | 1   | TinyUInt(1)                       |
| `0x09` | `50 00 00 00 40`     | 5   | Float32(2.0)                      |
| `0x0E` | `60 78`              | 2   | TinyString `"x"`                  |
| `0x10` | `70 03 08 09 0E`     | 5   | array leaf, w=0, n=3              |
| `0x15` | `80 01 06 10`        | 4   | object leaf, w=0, n=1             |

**Resolving `a[1]`**:

1. Read trailer -> root offset `0x15`.
2. At root offset, read object leaf; binary-search keys.
3. Dereference key offset `0x06` -> TinyString `"a"` matches.
4. Follow value offset `0x10` -> array leaf; pick slot 1 -> offset `0x09`.
5. Read `Float32(2.0)` at `0x09`.

---

## 8. Container Tree Invariants

Conforming files MUST satisfy, and readers MUST verify as they traverse:

1. **Type consistency.** JSON arrays use `0x70..0x77` or `0x33`; JSON objects
   use `0x80..0x87` or `0x34`. Pointers via array-child or object-value slots
   MAY land on any value tag.
2. **Internal node totals.** `total = Σ sub_total`. Each `sub_total` equals
   the referenced child's `n` (leaf) or `total` (internal).
3. **Minimum fanout.** `m ≥ 2` for every internal node. A single-node
   container is stored as a leaf, not wrapped.
4. **Object leaf ordering and uniqueness.** `keys[i] < keys[i+1]` by raw
   UTF-8 bytes; no key repeats within a leaf.
5. **Object cross-run duplicates: last-wins.** Within a logical object, the
   same key MAY appear in more than one run (one match per run, since
   leaves are themselves duplicate-free per invariant 4). When a lookup
   observes multiple matches, the match from the most recently flushed run
   is authoritative; earlier matches are shadowed. Readers MUST resolve in
   flush-time order and MUST NOT reject on multiple matches alone.
6. **Object internal fence consistency.** Each child's `key_off_lo` and
   `key_off_hi` MUST be `≤` and `≥` (UTF-8 byte order) every key in the
   child's subtree. Children stay in flush-time order, oldest first; sibling
   ranges MAY overlap subject to invariant 5.
7. **Offset well-formedness.** Every offset MUST be `≥ 6` (past the header),
   strictly less than the carrying node's position (postorder writes
   children first), and land on a valid type-code byte. Readers MUST reject
   otherwise.
8. **Offset width sufficiency.** Declared `W` MUST hold every offset in the
   node. Writers MUST pick the smallest `W` that fits; readers MUST accept
   any `W` that fits.

For invariants 2 and 3, readers SHOULD reject but MAY proceed if the
violation doesn't affect the current lookup - these are advisory traversal
metadata; bit-flip detection is out of scope (§1).

---

## 9. Body and Postorder Layout

The body is written in **postorder**: each child is fully written before its
referencing container. Consequences:

- The root has the highest offset in the body.
- All offsets in container nodes point earlier in the file than the node
  itself.
- No forward references; any offset a reader sees points to already-written
  bytes.

The body has no framing, length prefixes, or padding. Value sizes are
discovered structurally - scalars by fixed or length-prefixed payloads,
container nodes by their `n`/`m` counts. Selective retrieval follows offsets
from the root rather than scanning.

---

## 10. Header and Trailer

### 10.1 Header

The header is exactly **6 bytes** at the start of the file:

```
Offset  Size   Field     Description
------  ----   -----     -----------
 0       4     magic     ASCII "KAHN" (0x4B 0x41 0x48 0x4E)
 4       1     version   Format version (uint8), currently 0x01
 5       1     flags     Reserved, MUST be 0x00
```

Readers MUST reject on magic mismatch, unsupported version, or any flag bit
set.

### 10.2 Trailer

The trailer is exactly **12 bytes** at the end of the file:

| Offset from end | Absolute offset  | Size | Field         | Description                                        |
|-----------------|------------------|------|---------------|----------------------------------------------------|
| -12             | `file_size - 12` | 8    | `root_offset` | `uint64_le`; absolute offset of root type-code byte |
|  -4             | `file_size - 4`  | 4    | `magic`       | ASCII `"KAHN"`                                     |

`root_offset` MUST be `≥ 6` (past the header) and `≤ file_size − 13` (the
root must end at or before the trailer). Readers MUST reject if the trailing
magic mismatches or `root_offset` is out of range.

---

## 11. Error Handling

### 11.1 Reader Errors

| Phase | Condition | Behavior |
|---|---|---|
| **Header**     | Leading magic ≠ "KAHN"                       | MUST reject |
| **Header**     | Unsupported version                          | MUST reject |
| **Header**     | `flags ≠ 0`                                  | MUST reject |
| **Trailer**    | Trailing magic ≠ "KAHN"                      | MUST reject |
| **Trailer**    | `root_offset < 6` or `> file_size − 13`      | MUST reject |
| **Decoding**   | Unknown type code in traversal               | MUST reject |
| **Decoding**   | Truncated or overlong varuint                | MUST reject |
| **Object**     | Leaf keys not strictly UTF-8-sorted          | MUST reject |
| **Object**     | Duplicate key in a leaf                      | MUST reject |
| **Object**     | Multiple matches across runs in a key lookup | accept; resolve last-flushed-wins (§7.4) |
| **Object**     | Observed key violates a child's fence        | MUST reject (at the observing lookup) |
| **Container**  | Internal node with `m < 2`                   | MUST reject |
| **Container**  | `total ≠ Σ sub_total` (internal node)        | SHOULD reject (advisory, §8) |
| **Container**  | `sub_total` mismatches child `n`/`total`     | SHOULD reject |
| **Offsets**    | Offset not strictly earlier than its node, in the header, or on a non-type-code byte | MUST reject |
| **Lookup**     | Non-minimal integer encoding                 | SHOULD accept (writer-only rule) |
| **Lookup**     | Non-minimal string encoding                  | SHOULD accept |
| **Lookup**     | `n = 0` leaf instead of singleton            | SHOULD accept |
| **Lookup**     | Non-minimal offset width                     | SHOULD accept (writer-only rule) |
| **Lookup**     | Array index out of bounds                    | not-found or error (impl-defined) |
| **Lookup**     | Key not found                                | not-found or error (impl-defined) |

### 11.2 Writer Errors

Writers MUST reject:
- invalid UTF-8;
- NaN or Infinity floats;
- integer-valued numbers outside `[-2^63, 2^64-1]`;
- non-integer tokens whose significant digits exceed binary64 (§5.3);
- structurally invalid input (e.g., unbalanced containers).

When input JSON contains the same key more than once in a single object,
writers SHOULD apply **last-wins** semantics: the latest occurrence's value
is encoded. Within a single buffered leaf this means overwriting an earlier
pair; across already-flushed runs, the writer MAY either rewrite (if the
earlier run is still mutable) or simply append a new run — readers resolve
the duplicate by flush order (§7.4). Writers MAY instead reject duplicate
input keys; either behavior is conforming.

Writers MUST also honor minimal encodings:

| Range/case                  | Form                       | Section |
|-----------------------------|----------------------------|---------|
| integers `[-16, 31]`        | `TinyNegInt` / `TinyUInt`  | §5.2    |
| string bytes `1..15`        | `TinyString`               | §5.4    |
| zero-length array/object    | `EmptyArray` / `EmptyObject` | §7.0  |
| container offset width      | smallest `W` that fits     | §7      |

Violators are non-conforming; readers SHOULD still accept the output (§11.1).

---

## 12. Disk Storage Recommendations

**Non-normative**: optional writer practices that improve page-cache
behavior on disk or in object storage. Files following this guidance remain
conforming under §1–§11; readers need no changes.

### 12.1 Background: why default layout is suboptimal on disk

Resolving `foo.bar[3].baz` is a chain of pointer chases from `root_offset`.
In memory each hop is free; on disk each cross-page hop may fault. Two
defaults amplify this:

- **Variable, small nodes.** Fanout caps entries, but on-disk size ranges
  ~130 B (array leaf, `W=1`) to ~2 KB (object internal, `W=8`). Neither
  aligns to a 4 KB page; small nodes underfill pages.
- **Cold trailer + cold root.** Bootstrap touches two pages: the 12-byte
  trailer at `file_size − 12`, then the root node hundreds of bytes earlier.

### 12.2 Page-aligned trailer region

Writers SHOULD pad the body so the trailer lies in the file's last 4 KB
page - i.e., `file_size ≡ 0 (mod 4096)`. Padding uses unreferenced scalars
(e.g., `Null` bytes); since readers locate values via offsets, filler is
invisible.

This puts the trailer in one page that the first `pread` fetches. Since
postorder writes the root last, the root and its immediate children
naturally sit in the same final page - bootstrapping the upper tree without
a second I/O.

Pad only when trailing-page savings exceed padding cost (files larger than a
few pages). Below 4 KB, padding wastes more than it saves.

### 12.3 Target node size

Writers SHOULD pick fanouts so each node is close to but doesn't exceed a
**target byte size** (recommended: 4096, matching a common OS page). This
replaces a fixed `fanout` knob with a size-driven policy.

### 12.4 Avoid splitting nodes across page boundaries

When a node ≤ 4 KB would straddle a 4 KB boundary, writers SHOULD pad
(filler as in §12.2) to push it to the next boundary. Nodes > 4 KB are
emitted unaligned; two pages is unavoidable.

A cheap follow-on to §12.3: once nodes are page-sized, alignment turns "one
fault per hop" from near-miss to guarantee. Padding cost is bounded by
`4 KB × (#internal nodes)` - small for target-sized nodes.

---

## 13. Appendix

### 13.1 Complexity

Let `B` = fanout, `n` = container element count, `d` = JSON depth.

**Write path.** Sequential append only; no seeks, no rewrites.

| Operation                     | Time        | Peak memory              |
| ----------------------------- | ----------- | ------------------------ |
| Append a scalar or string     | O(size)     | O(1)                     |
| Close a container of width n  | O(n)        | O(B · log_B n)           |
| Encode whole document         | O(doc size) | O(d · B · log_B n_max)   |

Bytes per container, with `s ∈ {1,2,4,8}` the offset slot size (§7):

| Level     | Array          | Object (extra 2 fences/child) |
|-----------|----------------|-------------------------------|
| Leaf      | `s·n`          | `2s·n`                        |
| Internal  | `(8 + s)·n / (B − 1)` | `(8 + 3s)·n / (B − 1)` |

(Internal terms summed across levels.) For small-to-medium documents where
offsets fit in 1–2 bytes, directory overhead is 2–8× smaller than worst-case
fixed-width.

**Read path.** Random reads over the file.

| Operation                       | Time                          | Peak memory            |
| ------------------------------- | ----------------------------- | ---------------------- |
| Array element `arr[i]`          | O(log_B n) node reads         | O(log_B n) nodes       |
| Object key `o["k"]`             | O(R_k · log_B n) node reads   | O(1) node at a time    |
| Path of `k` segments            | Σ per-segment cost            | O(k)                   |
| Full deserialization            | O(doc size), near-sequential  | O(d · B)               |

`R_k` = runs whose fence range covers the key. Typically 1 (non-streamed,
single sorted run); at most the total run count `R` (heavily streamed with
overlap). Both writer and reader memory are independent of document size;
reader latency is `O(R · log_B n)` worst-case, logarithmic when `R_k = 1`.

### 13.2 Encoder sketch

Per open JSON container, keep a frame with a **level stack**: level 0 holds
current leaf entries; level `k > 0` holds pending `(sub_total, node_off)`
pairs for height `k`. When a level hits fanout, flush it as a node and
cascade one entry up. On close, unwind bottom-up; the last remaining entry's
`node_off` is the container's root.

Arrays are textbook bulk-loaded B+trees, unsorted. Objects carry key bytes
beside offsets at level 0; before each leaf flush, the buffer is sorted and
any duplicate **within the buffer** is collapsed to its last occurrence
(last-wins, §11.2) — duplicates against already-flushed runs need no
special handling, since readers resolve them by flush order. After sorting,
the leaf's smallest/largest
`key_off`s become its fence and bubble up with `(sub_total, node_off)` -
upper levels merge children's fences (min of `key_off_lo`s, max of
`key_off_hi`s by UTF-8) and stay flush-time ordered.

Internal-node entries are appended only when the child flushes (so
`node_off` is final). Leaf-entry offsets are also known at append time.
Width selection happens once per node at flush: take the max offset, round
up to the smallest `W ∈ {1,2,4,8}` that fits, encode `w` in the low 2 bits
of the type code.

### 13.3 Decoder sketch

Validate the header; read the trailer for `root_offset`. Dispatch on the
type code at each cursor. The offset width is the low 2 bits of the node's
type code, applied to all of its offset slots.

**Array index**: descend array-internal nodes by accumulating `sub_total`
until the target falls inside a subtree, then index into the leaf.

**Object key**: in a leaf, binary-search by dereferencing each `key_off`. In
an internal node, recurse into every child whose `[key_off_lo, key_off_hi]`
fence covers the key. A conforming file yields at most one match; a second
match MUST cause rejection.

Full deserialization is an in-order traversal.

---

*End of specification.*

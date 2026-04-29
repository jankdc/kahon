# Kahon

A binary encoding for JSON, optimized for write-once / read-many workloads with selective path retrieval.

See [spec.md](./spec.md) for the format specification.

## Implementations

- [jankdc/kahon-rs](https://github.com/jankdc/kahon-rs) — Rust writer/encoder

## Conformance Vectors

Lives in `./conformance/*.json`. Each vector pairs a JSON input or constructed file with the exact bytes a
conforming implementation must produce or reject. Vectors live in four
manifests:

### Manifest schema

Each manifest is a JSON array of vectors. Common fields:

```json
{
  "id": "category/short-name",
  "description": "One-line summary",
  "spec_refs": ["§4", "§5.2"],
  "bytes_hex": "4B41484E 0100 ... 4B41484E"
}
```

Fields by manifest:

- **valid.json**: adds `json_text` (the JSON literal as a string - preserves
  textual distinctions like `1` vs `1.0`).
- **invalid.json**: adds `must_reject: true` and `reject_reason` (cites the
  invariant or rule violated).
- **tolerated.json**: adds `writer_compliant: false`,
  `reader_should_accept: true`, and (when applicable) `json_text` plus a
  `minimal_form_id` cross-reference to the conforming encoding in
  `valid.json`. Forward-compatibility entries (e.g., Extension codes) have
  no JSON correspondence and may set `json_text` and `minimal_form_id` to
  `null`.
- **writer-rejects.json**: omits `bytes_hex`. Adds `json_text`,
  `writer_must_reject: true`, and `reject_reason` (the §5/§11.2 rule
  violated).

`bytes_hex` is whitespace-tolerant; spaces are decorative. Strip
non-hex characters before parsing.

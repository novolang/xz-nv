# Changelog

All notable changes to xz-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `xzdec` — the load-bearing interface. A feed-and-drain state machine
  in the vocabulary zstd-nv, brotli-nv and bzip2-nv already publish, so
  a caller decompressing four formats writes the pump once. The
  dictionary limit is the decision it will not make for a caller: a
  block header's one byte can ask for 4 GiB and `xz -9` routinely asks
  for 64 MiB, which is a demand entirely invisible in the file's name.
  `index_of` and `decompress_block` are the random-access pair that
  make `.xz` different from `.gz`.
- `xzenc` — the encoder, declared for a later release. `XzSettings` is
  a value a caller builds once rather than six optional arguments, and
  `with_block_bytes` is the one that decides whether the output is
  seekable at all.
- `xzstream`, `xzblock`, `xzindex`, `xzvli` — the container. The flags
  are written twice because the format requires the copies to agree; a
  declared block size is binding and an absent one is not checked; the
  index stores UNPADDED sizes and `offsets_of` adds the padding back
  once, here; and a variable-length integer has exactly one legal
  spelling per number, which `decode` enforces.
- `xzfilter` — the chain and its four rules in one call. All eight
  branch converters are in, each with its published instruction
  alignment, because getting an alignment wrong corrupts the output
  silently. `lzma2_dictionary_bytes` is where a 64 MiB demand becomes
  visible.
- `xzlzma2`, `xzlzma`, `xzrange` — the coder. The dictionary and the
  probability array are the CALLER'S, sized by
  `xzfilter.lzma2_dictionary_bytes` and `xzlzma.probability_bytes`
  before anything is committed; `xzrange.decode_bit` takes one
  probability value and answers the adapted one, so this package never
  holds a model. The legacy `.lzma` container is read, and the README
  says it has no magic number and no checksum.
- `xzcheck` — the four integrity checks behind one value, with the
  sizes of the twelve RESERVED types answered too, which is what lets a
  decoder skip a check it cannot verify and still find the next block.
  CRC-64/XZ is this package's, with its catalogue check value asserted,
  because crc-nv 0.1.4 publishes no 64-bit algorithm.
- `xzerror` — twenty-five fault kinds, a position naming which of seven
  layers was running, and FOUR predicates rather than three:
  `is_unsupported` exists because the format is extensible and a file
  from a newer encoder is not a broken file.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the four API suites reaches `not implemented:
  xz-nv.<module>.<fn>`.
- **Three `@value` structs answer a flag rather than a `Result`** —
  `XzVli`, `XzRange`/`XzRangeStep` and `XzSymbol`. A `@value` struct
  cannot be a `Result` payload in v1, and each of these sits in a loop
  that runs tens of millions of times per megabyte, so boxing one would
  allocate on the path that never fails. `xzvli.decode_fault`,
  `xzrange.start_fault` and `xzlzma.symbol_fault` turn the flag into
  the fault the rest of the package speaks. `XzSymbol` carries an
  integer fault CODE for the same reason: a boxed field cannot live in
  an unboxed aggregate.
- **A toolchain defect was found and filed while writing this
  package**: naming an item that does not exist in a local module is an
  internal compiler error at codegen rather than a name error at
  typing, and the constant form does not say which name was not found.
  Nothing here works around it — the author's own name was wrong — but
  the diagnostic that would have said so is missing.

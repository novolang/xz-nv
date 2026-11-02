# Changelog

All notable changes to xz-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-10-13

**Every body, and both directions checked against the real `xz`.** The
0.0.1 interface is published over: there is not a `todo()` left under
`src/`, the twelve modules are implemented, and the package's own
decoder reads .xz files it did not write while the real `xz -d` reads
every stream the package's encoder writes.

### Added

- **The decoder**, end to end: the stream header and footer with their
  duplicated flags, block headers and their optional binding sizes, the
  index read from the tail, all four check types over the four block
  layouts the format allows, `.lzma` files, and concatenated streams
  with padding between them. `decompress`, `index_of` and
  `decompress_block` are the one-shot entry points; `feed` and `finish`
  are the pump, and `read_all<S: Read[e]>` is the pump bound to a
  caller's stream.
- **The encoder**, end to end: `settings`, `with_check`,
  `with_filters` and `with_block_bytes`; `push` and `close`; and the
  one-shot `compress` and `compress_with`. The match finder is one
  greedy hash chain of 65,536 heads and a 256 KiB window of
  predecessors, and a preset moves its depth and its `nice` length as
  well as the dictionary size the block header declares.
- **External verification in both directions**, which is what this
  release's correctness rests on rather than the package's own round
  trip. Inbound, 23 files produced by the real `xz` — 19 `.xz` files
  over all four check types, presets 0, 6 and 9, three filter chains
  (plain LZMA2, delta+LZMA2 and delta+x86+LZMA2) and streams of zero,
  one, two, six and fifteen blocks, and 4 legacy `.lzma` files over the
  properties bytes — decode byte for byte, and a flipped byte is
  refused at each of the 296 positions of the three smallest streams.
  Concatenated streams and an unimplemented check type are hand-built
  cases under `tests/` instead. Outbound, 51 streams the encoder wrote,
  over every check type and four shapes of content, all ten presets,
  four block sizes, twelve filter chains, five chunked writes and two
  megabyte payloads, are accepted by `xz` 5.4.5 `-d`, which answers the
  original bytes in every case. A second sweep of 38 streams — every
  check type, the ten presets, four block sizes, thirteen filter
  chains, five chunked writes and two megabyte payloads — was re-run
  against the tree as it ships and held 37 of 38, the exception being
  the RISC-V chain, whose filter id this host's `xz` does not offer
  (`Unsupported options`). `xz -l --robot` also reads the container
  back as the settings that asked for it — 49 blocks for 4096-byte
  blocks over 200,000 bytes, 7 for 32 KiB — and the check type `None`,
  `CRC32`, `CRC64` and `SHA-256` in turn.
- **Three boundaries asserted as behaviour and not as bytes.** The
  match finder reaches the whole window the block header declares: two
  copies of a full-period sequence, the second at a distance of 180,000
  bytes, come out as one copy plus ninety-six bytes, and the block
  states the preset's dictionary rather than the book that was kept. A
  chain of five filters is refused — the block header holds its count
  in two bits plus one, and the writer stores `count - 1` in them, so
  an encoder that took five would describe a body of five as a body of
  one. And a filter ID below the 2^62 the format reserves is
  `unsupported` and not `corrupt`: the two are one comparison apart,
  and a file from a newer implementation is not a damaged one.
- **New public surface, all of it additive:** `xzlzma.decode_symbol`
  answering a value struct so the decoder never allocates per symbol,
  `xzlzma.decode_symbol_props` for the properties byte a caller already
  has, `xzlzma2.XzLzma2Chunk.props_byte` (a chunk header has to be
  writeable to be read back), `xzlzma2.XzLzma2State.state` and `.reps`,
  `xzlzma2.carries_properties`, and `xzrange.normalize`.

### Changed

- **Four signatures take `var` where they took a copy.**
  `xzfilter.decode_in_place` and `.encode_in_place`,
  `xzlzma2.decode_chunk` and `xzlzma.reset_probabilities` write into a
  caller's buffer in place — for a 64 MiB dictionary that is the
  difference between reusing one buffer and allocating one per block.
  All four are 0.0.x signatures that never ran, so no caller exists to
  break.
- **`XzDecoder` gained five fields and `XzEncoder` two**, both private.
  The decoder's hold the byte counts its accessors answer; the
  encoder's hold the content taken but not yet compressed into a block
  and the index records for the blocks already written.
- **Doc corrections:** `xzlzma.read_alone_header` said the `.lzma`
  header has no check, which is true and does not say what it does
  have; `xzblock`'s module comment said four filters where the format
  allows eight converters and `check_chain` accepts four filters;
  `check_sizes` now states what its `Ok` value is.
- **Refusals added where the format has no room for the shape:** a
  reserved filter id in `xzfilter.props_ok` and a zero compressed size
  in a block header are both named rather than read past.

### Fixed

- **A block header no longer sizes the decoder's buffer.** A header that
  declared 2^40 output bytes made `feed` and `decompress_block` reserve
  a terabyte, even under `with_limit`. The reservation is now held to the
  dictionary limit, to what is left of the output limit, and, in
  `decompress_block`, to what the input in hand can expand to. A block
  that outgrows its reservation is decoded again into a doubled buffer.

### Known

- **The encoder is worse than `xz` and says so in two places.** On 500
  KB of this package's own sources, `compress_with` at preset 6 answers
  119,112 bytes and `xz -6` answers 85,352; at preset 9 the two are
  117,476 and 85,352, and even `xz -1` beats it at 100,508. That gap is
  the greedy parser: the reference costs every candidate against its
  own current model and this encoder takes the longest match it found.
  Closing it is a release's work of its own and nothing here pretends
  otherwise.
- **The match finder's window is capped at 256 KiB** whatever the
  preset asks for. A 64 MiB dictionary would be 32 MiB of chain tables
  and this release does not spend it; every distance written is inside
  the window, so a decoder is still bounded by what the header
  declares. `XZ_ENC_WINDOW_MOST` is one constant and one loop.
- **One thread.** `xz -T0` is a host concern and this package has no
  effects, so the blocks are sequential.
- **SHA-256 is the check that buffers.** CRC-32 and CRC-64/XZ update
  incrementally; the SHA-256 row on crypto-nv takes a whole buffer, so
  `xzcheck` holds a block's content until it can hash it at once. A
  stream with `--check=sha256` therefore costs one copy of the block.
- **CRC-32 is rebuilt per call.** `crc-nv` publishes
  `crc.crc32(bytes, seed)` over a whole buffer with no incremental
  state, so every block's CRC-32 starts from the table again. Correct,
  and the only reason it is not a fault.
- **RISC-V has no external verifier.** The other seven branch
  converters are checked byte for byte against what `xz` produced; the
  RISC-V one is checked against the algorithm as specified, by
  round-trip and by its table of vectors, because the `xz` on this
  machine is older than that filter. It is the one converter whose
  wrongness a foreign decoder could not catch here.
- **An RISC-V converter asked to start at an unaligned offset masks the
  address rather than the whole instruction.** The two differ only for
  a start offset no producer writes, and the package's own
  encoder/decoder pair agrees either way, so no stream is affected.
- **A `.lzma` file whose header declares its size has no external
  producer here.** `xz --format=lzma` writes the unknown-size form
  always, so the branch that reads a declared size is exercised by
  hand-built headers rather than by a file from the reference. A
  size-declared stream still ends in the range coder's flush byte, and
  that byte is asserted forced: every one of its 256 values but the one
  written leaves the coder unclean. The `.lzma` reader is not part of
  the round trip either: this package writes the `.xz` container only.
- **`xzlzma`'s probability array has 6,144 entries of headroom at the
  default `lc`/`lp`/`pb`.** That is `PROB_MODEL_REST`, and it is
  deliberate: the array is sized from `(0x300 << (lc + lp)) + 6144` so
  the tables above the literal part start at a fixed offset whatever
  the properties byte says.
- **A stream whose block declares a dictionary smaller than the
  distances it uses** is caught by the LZMA2 layer's own distance
  check, not by the window the decoder sized before reading the block.
  `xzdec` bounds the dictionary from the header, as the format says a
  decoder should; a file that then reads outside that window is
  refused when the copy is attempted.
- **`stream_done` keeps the stream header.** A finished stream answers
  its check type and its flags rather than forgetting them, which means
  `check_type` survives the footer — the decision is stated in
  `xzdec`'s comment on it.
- **A toolchain defect is kept out of the package rather than worked
  around in it.** Nothing under `src/` or `tests/` subscripts a `Str`:
  a character read with `s[i]` leaks the string it answers, and inside
  an isolated run it leaves every verdict green while the run exits 23.
  The scratch corpus drivers walked filenames one character at a time
  and hit it; they read characters with `str.slice(name, i, i + 1)`.
  The filing is
  `novo/memory-perceus/a-character-read-with-a-str-subscript-leaks-the-string-it-answers`.
- **Three `@value` structs still answer a flag rather than a `Result`**
  — `XzVli`, `XzRange`/`XzRangeStep` and `XzSymbol` — for the reason
  0.0.1 recorded: a `@value` struct cannot be a `Result` payload in v1
  and each sits in a loop that runs millions of times per megabyte.

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

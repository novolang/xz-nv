# xz-nv

`.xz` is a container format for compressed data, defined by
[The .xz File Format specification](https://tukaani.org/xz/xz-file-format.txt)
and implemented by [XZ Utils](https://tukaani.org/xz/). What it usually holds
is LZMA2, a chunked form of the Lempel-Ziv-Markov chain algorithm. This package
reads and writes both in novo-lang, and performs no input or output itself.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What xz is

An `.xz` **file** is one or more *streams*, separated by zero padding to a
four-byte boundary. Concatenation is the format's own multi-file convention.

A **stream** is a twelve-byte header, a sequence of blocks, an index, and a
twelve-byte footer. The header and the footer both carry the same two *stream
flag* bytes, and the format requires the two copies to agree — which is what
catches a file that was truncated and had another file's tail appended. Both
also carry a CRC-32 over those flags, so a decoder can tell "this is not xz"
from "this is xz and its first twelve bytes are damaged".

A **block** is a header, compressed data, padding, and an *integrity check*.
The header may declare the block's compressed size, its uncompressed size, both
or neither; where a field is present, the decoder must check it. A block whose
header declares both can be decoded on its own, in parallel with its
neighbours.

A block's header also carries its **filter chain**: at most four filters,
exactly one of them a compression filter, and that one last. A chain is applied
in order on the way out and in reverse on the way in.

- **LZMA2** is the only compression filter the format defines. It is a sequence
  of chunks, each with its own size: either up to 64 KiB of bytes stored as they
  are, or up to 2 MiB of output from a compressed payload. A chunk's control
  byte says which of four things were reset before it — nothing, the LZMA
  state, the state and the properties, or the state, the properties and the
  dictionary.
- **LZMA** itself is LZ77 with an adaptive binary model. The output is literals
  and matches, and every decision is a range-coded bit whose probability depends
  on a *context*: which of twelve states the machine is in, some high bits of
  the previous byte, some low bits of the output position. It keeps the last
  four match distances and can name one instead of coding a new one.
- **The delta filter** replaces each byte with its difference from the byte a
  fixed distance behind it. **The eight branch converters** — x86, PowerPC,
  IA-64, ARM, ARM-Thumb, SPARC, ARM64, RISC-V — rewrite an instruction's
  relative jump offsets to be absolute. Neither compresses anything. Both make
  what follows them compress much better.

The **index** sits after the last block and holds one record per block: the
block's *unpadded* size — its header plus its compressed data plus its check,
without the block padding — and its uncompressed size. The stream footer says
how large the index is, so a reader that can seek takes the last twelve bytes,
jumps back, and has a table of contents without decompressing anything.

| Value | Size |
| --- | --- |
| Stream header, and stream footer | 12 bytes each |
| Block header | 8 to 1024 bytes |
| Filters in a chain | at most 4 |
| Integrity checks | none (0 bytes), CRC-32 (4), CRC-64 (8), SHA-256 (32) |
| Variable-length integer | 1 to 9 bytes, ceiling 2^63 - 1 |
| LZMA2 uncompressed chunk | at most 65,536 bytes |
| LZMA2 compressed chunk output | at most 2,097,152 bytes |
| LZMA dictionary | 4 KiB to 4 GiB; `xz -9` asks for 64 MiB |
| LZMA probability model | 14,134 entries at the default `lc=3, lp=0` |
| LZMA match length | 2 to 273 bytes |

## Install

```
novo pkg add xz-nv
```

## Example

```novo
use std.bytes
use xzdec

fn main() [io]
    // A whole `.xz` held in memory.  `Ok` means every block check,
    // every index and every stream footer verified.
    match xzdec.decompress(bytes.zeros(12))
        Err(f) => println(f.message())
        Ok(b)  => println("${bytes.len(b)} byte(s)")
```

Build and test with:

```
novo pkg build                  # type- and effect-check the package
novo test --isolate tests/xzdec_tests.nv
```

Today `novo test` fails on purpose: every assertion reaches
`not implemented: xz-nv.<module>.<fn>`.

## What the package contains

| Module | Contents |
| --- | --- |
| `xzdec` | The decoder: `feed` and `finish`, the two limits, the one-shot `decompress`, `index_of` and `decompress_block` for random access, `read_all` over a stream you supply. |
| `xzenc` | The encoder: the settings value, `push` and `close`, the one-shot `compress`, and `compress_bound`. |
| `xzstream` | The stream header and footer, the flags, and the padding between streams. |
| `xzblock` | The block header, its optional binding sizes, and the unpadded size it contributes to the index. |
| `xzindex` | The index: its records, the totals, the block offsets, and the check against the blocks that were read. |
| `xzfilter` | The filter chain: delta, the eight branch converters, LZMA2's dictionary byte, and the chain's four rules. |
| `xzlzma2` | The chunk layer: control bytes, the four resets, and a decoder over the caller's dictionary and probability array. |
| `xzlzma` | LZMA1: the properties byte, the twelve-state machine, the length and distance codes, the four rep distances, and the legacy `.lzma` container. |
| `xzrange` | The binary range decoder, as two integers and a caller's probability array. |
| `xzcheck` | The four integrity checks, and CRC-64/XZ, which no package on this registry published. |
| `xzvli` | The variable-length integer every size field is, and the four-byte padding rule. |
| `xzerror` | `XzFault` with the block, layer and byte offset it happened at, and the four questions `is_corrupt`, `is_incomplete`, `is_limit` and `is_unsupported`. |

No function in this package opens a file, reads a clock or waits for anything.
The one function that meets a stream charges its caller rather than declaring
an effect of its own:

```novo ignore
pub fn read_all<S: Read[e]>(src: S) -> Result<Bytes, XzFault> [e]
```

A file charges `[io]`; an in-memory buffer charges nothing.

## How to choose an entry point

**You have a whole `.xz` in memory: `xzdec.decompress`.** It buffers, so its
`Ok` means every block's check, the index and the stream footer all verified.

**The file came from somewhere you do not trust:
`xzdec.decompress_bounded`.** The same call with both bounds.

**The bytes arrive in pieces: `xzdec.decoder`, `feed` and `finish`.**

```novo ignore
var d = xzdec.decoder()
let (d2, out) = xzdec.feed(d, chunk)      // as many times as you like
let tail = xzdec.finish(d2)!              // once, at end of stream
```

A chunk may split anything, and the decoder carries what it could not finish
into the next call. `feed` answers a new decoder rather than modifying one.

**You want a table of contents: `xzdec.index_of`.** It reads the last twelve
bytes and then the index, and decompresses nothing.
`xzindex.total_uncompressed` is what `xz --list` prints.

**You want one block out of the middle: `xzindex.block_at`,
`xzindex.offsets_of` and `xzdec.decompress_block`.** The first says which block
an output offset falls in, the second where it starts, and the third decodes
it.

**You are decoding LZMA2 into memory you own: `xzlzma2.decoder`.**
`xzfilter.lzma2_dictionary_bytes` and `xzlzma.probability_bytes` answer the two
buffer sizes first.

**You have a legacy `.lzma` file: `xzlzma.decompress_alone`.** Read rule 7
first.

**You are producing an `.xz`: `xzenc`.** Read "What the encoder does" first.

## The rules a user needs

1. **The dictionary is the demand you cannot see in the file's name.** A block
   header's one dictionary byte can ask for 4 GiB, and `xz -9` routinely asks
   for 64 MiB. `xzdec.decoder()` allows 64 MiB, so the ordinary file decodes
   and a hostile one does not; a block asking for more is `XzLimitReached`
   carrying both numbers, so a caller can raise `xzdec.with_dict_limit`
   deliberately and retry.
2. **The output bound is a second, separate decision, and it is off by
   default.** An `.xz` usually declares its uncompressed size — in every block
   header that carries the field, and in every index record — so a caller who
   wants to know asks `xzdec.index_of` and gets an answer without decompressing
   a byte. `xzdec.with_limit` is there for the cases where that is not enough.
3. **Four questions, not one.** `xzerror.is_corrupt` says the data is damaged,
   `xzerror.is_incomplete` says more bytes would help, `xzerror.is_limit` says
   this program set a bound the data exceeded, and `xzerror.is_unsupported`
   says the file is good and asks for something this package does not do. The
   last one exists because the format is extensible: twelve check types and
   many filter IDs are reserved, and a decoder that called a file from a newer
   encoder "corrupt" would be telling a user their archive was broken when it
   was not.
4. **An unsupported check is skipped, not refused.** The format fixes the sizes
   of the check types it reserves, so a decoder can step over one it cannot
   verify and still find the next block header. `xzdec.with_strict_check` turns
   that into a refusal for a caller who would rather not read data that nothing
   verified. A stream may also declare check type `none`, in which case nothing
   verified it at all — `xzdec.check_type` is how a caller sees that.
5. **A size a block header declares is binding.** Where the field is present,
   a block that produced or consumed a different number is `XzSizeMismatch`.
   Where it is absent the decoder finds the end from the filter chain.
6. **The index is checked against the blocks.** Every block's integrity check
   can pass and the file still be wrong, because the index also fixes the order
   and the count of the blocks. `xzindex.agrees_with` is that comparison, and
   `xzdec.finish` runs it.
7. **A `.lzma` file has no magic number and no checksum.** It cannot be
   recognised by inspection and a damaged one is not detectable at all; its
   size field may say "unknown", in which case nothing in the file says how
   large the content is. `.xz` exists to fix both. A caller with a choice reads
   `.xz`, and a caller reading `.lzma` passes a bound.
8. **Content up to the last block boundary has already been verified.** `.xz`
   checks every block, not only the stream. `xzdec.verified_out` is that count
   and `xzdec.total_out` is everything produced.
9. **A variable-length integer has exactly one legal spelling per number.** The
   format requires the shortest form, and `xzvli.decode` refuses anything else
   — because a block's size appears twice in a file and a number with several
   spellings is a number two decoders can disagree about.
10. **Padding is checked, never skipped.** Non-zero padding is a corrupt file,
    in all three places padding appears.
11. **Every public type, variant and module name in this package starts `Xz`
    or `xz`.** Type and variant names are unique across a whole program,
    dependencies included, so two packages that both declared `Block` could not
    be used together. `Block`, `Index`, `Filter`, `Check` and `Fault` are all
    names another package will want, so this one takes none of them.

## What the encoder does

The decoder has an oracle: the format specification says what a stream means,
and XZ Utils' test files say whether this one agrees. An encoder has no such
target. Any stream the reference decoder reads is a *correct* stream; whether
it is a *good* one is a question about ratio, which XZ Utils answers with a
binary-tree match finder, an optimal parser that costs every candidate against
the adaptive model's own current probabilities, and a price table recomputed as
the model moves.

The promise is therefore bounded, and the bounds are in the signatures.

| | |
| --- | --- |
| Threads | one. Nothing in this package spawns anything. |
| Match finder | one greedy hash chain. Not a binary tree, and no optimal parsing. |
| Presets | the nine levels map to a dictionary size and a chain depth, and nothing else. `xz -9e`'s extra pass has no counterpart. |
| Filters | all ten, both directions. A filter is reversible arithmetic with no ratio question in it. |
| Container | `.xz` only. The legacy `.lzma` container is read and never written. |

A caller asking for preset 9 gets a valid `.xz` that is measurably larger than
`xz -9`'s. `XzSettings` is a value a caller builds once and names rather than
six optional arguments, and `xzenc.with_block_bytes` is the one that matters
most: a single-block `.xz` has an index with one record in it, and a file
written in 16 MiB blocks can be decoded from any of them.

## What is not included

- **`lzip` and the `.7z` LZMA container.** Both carry LZMA, and neither is this
  format. A package for either is a different row.
- **Parallel decoding.** Every piece it needs is here — `xzindex.offsets_of`
  and `xzdec.decompress_block` decode one block with no knowledge of the blocks
  before it — but running them at once is scheduling, which is a host effect,
  and this package has none.
- **A build for a microcontroller with no heap allocator.** Even preset 0 asks
  for a 256 KiB dictionary, on top of a 28 KiB probability array.
- **Writing the legacy `.lzma` container.** Read, never written.
- **An optimal parser.** See "What the encoder does".

## Related packages

- [zstd-nv](https://novo-lang.org/packages/zstd-nv),
  [brotli-nv](https://novo-lang.org/packages/brotli-nv) and
  [bzip2-nv](https://novo-lang.org/packages/bzip2-nv) publish the same
  feed-and-drain vocabulary for their formats, so a caller decompressing four
  formats writes the pump once.
- [flate-nv](https://novo-lang.org/packages/flate-nv) is DEFLATE, gzip and
  zlib.
- [crc-nv](https://novo-lang.org/packages/crc-nv) supplies the CRC-32 this
  format checksums its headers with. Its CRC-64 row does not exist, so
  `xzcheck` owns CRC-64/XZ.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies the SHA-256 an
  `.xz` block may carry as its check.

## The reference implementation

[XZ Utils](https://tukaani.org/xz/), with
[the .xz file format specification](https://tukaani.org/xz/xz-file-format.txt)
and [the LZMA specification](https://tukaani.org/xz/lzma-file-format.txt) as
the format documents. This package keeps the reference's own names where they
are good — `lc`, `lp`, `pb`, `unpadded_size`, the four reset kinds — so a
reader with that source open recognises what they are.

## Test vectors

XZ Utils' own `tests/files` directory is the oracle. It ships files built to
exercise each of the four check types, each filter, an unsupported check type,
a truncated stream, a stream with several blocks, a concatenated file, and a
set of deliberately malformed files that a decoder must refuse. When the bodies
land, every file in it is decoded or refused as its name says, as a generated
run beside the four suites in `tests/`.

Two check values have their own oracle and are asserted in
`tests/xzcontainer_tests.nv`: CRC-64/XZ of the nine bytes `123456789` is
`0x995DC9BBDF1939FA`, and the properties byte `0x5D` is `lc=3, lp=0, pb=2`,
which is what every `xz` and every `lzma` writes unless told otherwise.

## Implementation status

Every function is a `todo()`. Every type is declared.

| Module | Implemented |
| --- | --- |
| `xzdec` — `XzDecoder` and its twenty-two functions | no |
| `xzenc` — `XzSettings`, `XzEncoder` and their thirteen functions | no |
| `xzstream` — `XzStreamFlags`, `XzStreamHeader`, `XzStreamFooter` and their eight functions | no |
| `xzblock` — `XzBlockHeader` and its nine functions | no |
| `xzindex` — `XzIndexRecord`, `XzIndex` and their twelve functions | no |
| `xzfilter` — `XzFilter` and its thirteen functions | no |
| `xzlzma2` — `XzLzma2Chunk`, `XzLzma2State` and their eleven functions | no |
| `xzlzma` — `XzLzmaProps`, `XzRepDistances`, `XzAloneHeader`, `XzSymbol` and their twenty-one functions | no |
| `xzrange` — `XzRange`, `XzRangeStep` and their nine functions | no |
| `xzcheck` — `XzCheck` and its thirteen functions | no |
| `xzvli` — `XzVli` and its eight functions | no |
| `xzerror` — `XzWhere`, `XzFaultKind`, `XzFault` and their eight functions | no |

## Licence

Apache-2.0. See `LICENSE`.

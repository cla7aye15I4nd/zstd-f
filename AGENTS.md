# Working rules

Goal: kernel-checked Lean 4 proofs about a faithful translation of upstream Meta Zstandard (`dev`),
covering the standalone C decoder, dictionary loader, frame inspector and shared entropy/checksum
units (`lib/zstd.h`, `lib/zstd_errors.h`, `lib/decompress/*.c`, and the six `lib/common/*.c` units
linked by the decoder).

The goal is full safety (no undefined behaviour, no memory error, no dangling dictionary/prefix
borrow, no memory leak, and termination) of every public decoder, dictionary, frame-inspection and
error/version function in `lib/zstd.h`, across every input and every sequence of calls, for the
portable scalar x86_64 Linux LP64 build:

```text
-DZSTD_DISABLE_ASM -DDYNAMIC_BMI2=0 -DZSTD_LEGACY_SUPPORT=0
-DXXH_NAMESPACE=ZSTD_ -DXXH_NO_XXH3 -DXXH_VECTOR=0
```

Specifically:

1. **Streaming decompression (`ZSTD_decompressStream`)** is safe on every input and every split of
   input and output buffers across calls, including multi-frame and skippable-frame streams, raw
   and formatted dictionaries, ring-buffer / contiguous / non-contiguous (`extDict`) output modes,
   allocation failures, and repeated calls after frame completion or error;
2. **One-shot and context decompression** (`ZSTD_decompress`, `ZSTD_decompressDCtx`,
   `ZSTD_decompress_usingDict`, `ZSTD_decompress_usingDDict`) is safe on every input;
3. **Every other public decoder, dictionary, frame-query and utility function** in `lib/zstd.h` is
   safe across every valid call sequence and cleanly refuses invalid parameters as specified:
   - Context & stream lifecycle: `ZSTD_createDCtx`, `ZSTD_freeDCtx`, `ZSTD_sizeof_DCtx`,
     `ZSTD_createDStream`, `ZSTD_freeDStream`, `ZSTD_initDStream`, `ZSTD_sizeof_DStream`,
     `ZSTD_DStreamInSize`, `ZSTD_DStreamOutSize`, `ZSTD_dParam_getBounds`,
     `ZSTD_DCtx_setParameter`, `ZSTD_DCtx_reset`;
   - Dictionaries & prefixes (both raw content and `0xEC30A437` formatted dictionaries with
     pre-built Huffman/FSE tables): `ZSTD_createDDict`, `ZSTD_freeDDict`, `ZSTD_sizeof_DDict`,
     `ZSTD_DCtx_loadDictionary`, `ZSTD_DCtx_refDDict`, `ZSTD_DCtx_refPrefix`,
     `ZSTD_getDictID_fromDict`, `ZSTD_getDictID_fromDDict`, `ZSTD_getDictID_fromFrame`;
   - Frame inspection & library queries: `ZSTD_getFrameContentSize`, `ZSTD_getDecompressedSize`,
     `ZSTD_findFrameCompressedSize`, `ZSTD_decompressBound`, `ZSTD_isError`, `ZSTD_getErrorCode`,
     `ZSTD_getErrorName`, `ZSTD_versionNumber`, `ZSTD_versionString`,
     `ZSTD_isDeterministicBuild`.

The compressor (`lib/compress/`), dictionary trainer (`lib/dictBuilder/`), legacy v0.1–v0.7 decoders
(`lib/legacy/`), handwritten amd64 assembly (`huf_decompress_amd64.S`), and full RFC 8878 functional
decoding equivalence beyond what safety requires are not planned: any functional fact that a safety
proof depends on (such as FSE state bounds, Huffman jump-table and bit-consumption bounds, or
`WILDCOPY_OVERLENGTH` sequence margin invariants) is in scope.

Work as fast as honest work allows: process exists to keep the result honest, not to slow it down.

## Source

- `zstd/` is the upstream submodule (`https://github.com/facebook/zstd`). Never modify it. The
  verified source is `zstd/` plus `patches/*.patch` applied in name order.
- The verified C translation units are the exact ten `.c` files of the standalone decoder (plus the
  headers they include from `lib/`, `lib/common/`, and `lib/decompress/`):
  - `lib/common/debug.c`, `lib/common/entropy_common.c`, `lib/common/error_private.c`,
    `lib/common/fse_decompress.c`, `lib/common/xxhash.c` (`XXH64` path),
    `lib/common/zstd_common.c`;
  - `lib/decompress/huf_decompress.c`, `lib/decompress/zstd_ddict.c`,
    `lib/decompress/zstd_decompress.c`, `lib/decompress/zstd_decompress_block.c`.
- A patch is allowed only for a confirmed Zstandard bug (or C standard undefined behaviour reachable
  through the documented API) that blocks a proof. It must be minimal, match a fix submitted
  upstream, be listed in `patches/README.md`, and have a corresponding ablation case in
  `tests/patch-reverts/<case>/` showing that reverting the patch breaks the proof and reproduces on C
  (under ASan/UBSan or a C probe).
- Translations mirror the C layout:
  - `zstd/lib/common/X.c` is `ZstdF/Zstd/common/X.c.lean` (`.h` -> `.h.lean`);
  - `zstd/lib/decompress/X.c` is `ZstdF/Zstd/decompress/X.c.lean` (`.h` -> `.h.lean`);
  - `zstd/lib/zstd.h` and `zstd/lib/zstd_errors.h` are `ZstdF/Zstd/zstd.h.lean` and
    `ZstdF/Zstd/zstd_errors.h.lean`.
  Each function keeps its C name, and its docstring starts with the C signature.
- Specifications live in `ZstdF/Spec/{Common,Safety,Functional}/`. No `Safety` or `Common` module
  may import a `Functional` module, and no `Spec` module may import `ZstdF/Proof/`.
- Proofs about `X.c` live in `ZstdF/Proof/X.c.<topic>.lean`. Proofs of public goals live in
  `ZstdF/Proof/public-<name>.lean`.
- Heap-free mathematics (reverse bitstream geometry, FSE/tANS table invariants, Huffman X1/X2 table
  bounds, XXH64 arithmetic, sequence-offset window algebra) goes to `ZstdF/Math/`.
- The runtime (heap, pointers, unaligned little-endian memory reads/writes `MEM_read*`/`MEM_write*`,
  loops, faults) is `ZstdF/Runtime/*.lean`.

## Translation

- Translate statement by statement. Do not reorder, merge or simplify.
  Four normalisations are accepted and are not translation bugs:
  an out-parameter of an internal function is passed by value and returned as a second result;
  two reads of one location with no store between them are one read;
  consecutive assignments to fields of one record are one record update;
  a signed operation that a preceding test of the same function keeps in range uses the plain
  operator. Anything else is a translation bug, documented with examples in
  `docs/en/design/translation.md`.
- Every undefined behaviour the C can reach raises `Fault.ub`. Unsigned arithmetic wraps. Signed
  arithmetic, shifts and narrowing conversions are checked. Pointer arithmetic (`p + k`, `p - k`,
  `p - q`) and relational pointer comparisons (`<`, `<=`, `>`, `>=`) are checked against single-object
  allocation bounds at the point they are evaluated, matching C11 §6.5.6 and §6.5.8.
- Reading an uninitialised integer returns an arbitrary value from `Heap.garbage`, and a theorem
  must hold for every garbage function. A pointer in fresh memory is `NULL` in the model, which is
  weaker than C: the proof and review show that every pointer field of `ZSTD_DCtx` and `ZSTD_DDict`
  is written before it is read (`docs/en/trusted-base.md`).
- Never change a translation to make a proof easier. A mismatch with the C is a translation bug.
  Fix it, and add a differential test that catches it.
- Every proof depends on the runtime. Change `ZstdF/Runtime/` only for a confirmed mismatch with C,
  in its own commit.

## Key Zstandard hazards to model and verify without shortcuts

1. **Fast sequence execution and `WILDCOPY_OVERLENGTH` (`zstd_decompress_block.c`).**
   `ZSTD_wildcopy` intentionally copies in 8- or 16-byte chunks up to `WILDCOPY_OVERLENGTH` (32
   bytes) past the logical end of a literal or match copy, relying on fast-loop guards
   (`oend_w = oend - WILDCOPY_OVERLENGTH`) before falling back to `ZSTD_execSequenceEnd` /
   `ZSTD_safecopy`. In `SplitLitBuffer` mode, decoded literals temporarily reside at the tail of the
   destination buffer itself while sequences overwrite the front. Prove from first principles that
   every wildcopy stays strictly within the writable destination block and never clobbers unread
   split literals.
2. **Cross-object `virtualStart` and `extDict` pointer arithmetic (`zstd_decompress.c`,
   `zstd_decompress_block.c`, `compiler.h`).**
   When output buffers are non-contiguous across calls or a dictionary/prefix is attached, upstream C
   computes `virtualStart = prefixStart - extDictSize` (`ZSTD_wrappedPtrSub` or raw `char*`
   subtraction) and `oLitEnd - virtualStart` (`ZSTD_wrappedPtrDiff` or raw subtraction) across
   distinct heap objects. Do **not** weaken `ZstdF/Runtime/Ptr.lean` to allow cross-object pointer
   subtraction or out-of-object pointer formation: investigate every site on real C (with UBSan and
   ASan `detect_invalid_pointer_pair=2`), record confirmed C11 undefined behaviour in
   `docs/en/findings.md`, and isolate minimal `uintptr_t` / size-arithmetic fixes in `patches/` with
   `tests/patch-reverts/` reproductions.
3. **Reverse bitstreams (`BIT_DStream_t`), 4-stream Huffman (`huf_decompress.c`) and FSE tables
   (`fse_decompress.c`, `entropy_common.c`).**
   `BIT_initDStream` and `BIT_reloadDStream` read backwards from `srcEnd` toward `src`; 4-stream
   Huffman splits the compressed buffer via a 6-byte jump table (`ip1..ip4`); and
   `ZSTD_loadDEntropy` aliases the three adjacent FSE table arrays (`LLTable`, `OFTable`, `MLTable`)
   inside `ZSTD_entropyDTables_t` as a flat `U32` workspace while building `hufTable`. Model these
   exact buffer/workspace layouts without unsound casts or unproved table-entry bounds.
4. **Formatted dictionaries (`0xEC30A437`) and borrowed references (`ZSTD_DCtx_refDDict`,
   `ZSTD_DCtx_refPrefix`).**
   Prove full parser safety for untrusted formatted dictionaries (`ZSTD_createDDict`,
   `ZSTD_DCtx_loadDictionary`), including initial `rep[0..2]` offset validation against dictionary
   size, and track caller-buffer liveness explicitly in the sequence invariant whenever `refDDict` or
   `refPrefix` borrows external memory across `ZSTD_decompressStream` calls.

## What counts as proved

`ZstdF/Results.lean` is the single Lean-checked registry of results: it pairs each proved `Goal.*`
with its theorem in `proved` and lists unfinished goals in `notDone`. Building `ZstdF` checks in Lean
itself that:
1. every theorem's type is definitionally/syntactically identical to its `Goal.*` declaration;
2. only `propext`, `Classical.choice` and `Quot.sound` are used (verified on the compiled environment
   alongside `PartialCheck.lean`);
3. every `Goal.*` under `ZstdF/Spec/` appears in `proved` or `notDone`, and `public-api.json` covers
   every public decoder/dictionary/query declaration of `lib/zstd.h`.

Rules that hold everywhere:

- Never use `sorry`, `native_decide`, `bv_decide`, `axiom`, `implemented_by`, `extern`, `unsafe`,
  `partial` or `opaque`. Do not look for ways around the checks; report the obstacle.
- Every safety goal is paired with a non-vacuity witness theorem showing all premises hold
  simultaneously on a concrete heap and non-trivial execution.
- Every precondition narrower than `zstd.h` allows (such as pairwise disjoint caller buffers or the
  default `malloc`/`free` allocator when `customMem` is zeroed) is stated explicitly in the goal and
  listed in `docs/en/deviations.md`. Never shrink a domain silently.
- **No callee contracts as assumptions.** Every helper's contract is a theorem about the translated C
  callee. Representation invariants (`DCtxInv`, `DDictInv`) are established by `ZSTD_createDCtx` /
  `ZSTD_createDDict` and preserved across every public operation.
- Differential tests (comparing Lean executions against ASan+UBSan-instrumented C across single-shot,
  1-byte streaming splits, `extDict` buffer switches, corrupted FSE/Huffman tables, and formatted
  dictionaries) are evidence of fidelity, not proof. Say so when reporting.

## Proof workflow

1. **Bootstrap runtime and CI gates first.** Set up `ZstdF/Runtime/`, `ZstdF/Results.lean`,
   `PartialCheck.lean`, `check-sorry.sh`, `check-spec-layers.py` and `verification.py` so every
   commit passes the build and audit checks from day one.
2. **Translate all ten `.c` units and run C-vs-Lean differential tests before proving.** Shake out
   translation and C11 pointer-UB issues early with differential tests and sanitizers.
3. **Test goal statements on concrete C executions.** Verify behaviour on `NULL` with zero capacity,
   allocation failure at every `ZSTD_customMalloc` site, and repeated calls after errors.
4. **Build skeletons top down (`skeleton/<result>`).** `sorry` is allowed only on `skeleton/*`
   branches. Keep the open-lemma list in the skeleton header comment, not in `docs/`.
5. **Hide `DCtxInv` and `DDictInv` behind small interface lemmas** so changing an internal clause
   rebuilds only its interface module. Build with at most `-j 4` under a 16 GB memory limit, and
   delete unused modules outside the dependency closure of `ZstdF/Results.lean`.

## Findings and documents

- List confirmed upstream Zstandard defects reachable through documented `zstd.h` calls in
  `docs/en/findings.md` and `docs/zh/findings.md`, with sanitizer/probe reproductions under
  `tests/regression/<name>/` and patch-ablation tests under `tests/patch-reverts/<name>/`.
- English and Chinese documentation sources live under `docs/en/` and `docs/zh/` at mirrored
  lowercase-hyphenated paths, with no `.en.md`, `.zh.md` or subdirectory `README.md` files.
  `docs/en/progress.md` (paired with `docs/zh/progress.md`) is the sole status page (`done` or
  `not done`).
- Do not put machine names, addresses, user names, local paths or tool names in the repository or in
  commit messages.
- Work lands via pull requests into `main` (or integration branches merged into `main` once CI is
  green). Parallel workers own disjoint files and never edit the same file concurrently.

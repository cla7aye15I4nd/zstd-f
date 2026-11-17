# zstd-f

Kernel-checked Lean 4 safety verification of Meta Zstandard's C decoder (`libzstd`:
`lib/zstd.h`, `lib/zstd_errors.h`, `lib/decompress/*.c`, and the `lib/common/*.c` units it links)
for the portable scalar x86_64 Linux LP64 build.

## Scope

Full safety — no undefined behaviour, no out-of-bounds read or write (including `ZSTD_wildcopy`
margins and `SplitLitBuffer` overlaps), no dangling dictionary or prefix borrow, no memory leak,
and termination — across every public decoder, dictionary, frame-inspection and error/version
function of `zstd.h` and every call sequence:

- **Streaming & context lifecycle:** `ZSTD_createDCtx`, `ZSTD_freeDCtx`, `ZSTD_sizeof_DCtx`,
  `ZSTD_createDStream`, `ZSTD_freeDStream`, `ZSTD_initDStream`, `ZSTD_sizeof_DStream`,
  `ZSTD_DStreamInSize`, `ZSTD_DStreamOutSize`, `ZSTD_dParam_getBounds`, `ZSTD_DCtx_setParameter`,
  `ZSTD_DCtx_reset`, `ZSTD_decompressStream`
- **One-shot & context decompression:** `ZSTD_decompress`, `ZSTD_decompressDCtx`,
  `ZSTD_decompress_usingDict`, `ZSTD_decompress_usingDDict`
- **Dictionaries & prefixes (raw and `0xEC30A437` formatted dictionaries):** `ZSTD_createDDict`,
  `ZSTD_freeDDict`, `ZSTD_sizeof_DDict`, `ZSTD_DCtx_loadDictionary`, `ZSTD_DCtx_refDDict`,
  `ZSTD_DCtx_refPrefix`, `ZSTD_getDictID_fromDict`, `ZSTD_getDictID_fromDDict`,
  `ZSTD_getDictID_fromFrame`
- **Frame inspection & queries:** `ZSTD_getFrameContentSize`, `ZSTD_getDecompressedSize`,
  `ZSTD_findFrameCompressedSize`, `ZSTD_decompressBound`, `ZSTD_isError`, `ZSTD_getErrorCode`,
  `ZSTD_getErrorName`, `ZSTD_versionNumber`, `ZSTD_versionString`, `ZSTD_isDeterministicBuild`

See `AGENTS.md` for working rules, target build flags, Zstandard-specific verification hazards, and
proof acceptance criteria.

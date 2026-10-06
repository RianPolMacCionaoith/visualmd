# VisualMD v1 format

VisualMD is a visual file-transport format intended for source code and Markdown. A v1 file is compressed,
split into equal-size data blocks, augmented with whole-block recovery blocks, and placed into independent QR
codes on a single sheet when practical.

## Processing pipeline

```text
original bytes
  -> SHA-256
  -> zlib level 9
  -> zero-pad to k equal chunks
  -> Cauchy GF(256) erasure code: k data + r recovery chunks
  -> CRC32 on each chunk
  -> Base45 payload
  -> QR alphanumeric mode, ECC M
  -> PNG sheet + human-readable checksums
```

All sizes in QR headers are decimal ASCII. Hashes and CRCs are uppercase hexadecimal.

## QR payload grammar

A QR contains one ASCII record:

```text
VMD1:<file_id>:<index>:<k>:<r>:<chunk_size>:<compressed_size>:<original_size>:<sha256>:<suffix>:<filename_b32>:<crc32>:<payload_b45>
```

Fields:

- `file_id`: first 12 hexadecimal characters of the original-file SHA-256.
- `index`: zero-based block index. `0..k-1` are systematic data blocks; `k..k+r-1` are recovery blocks.
- `k`: number of data blocks.
- `r`: number of recovery blocks.
- `chunk_size`: decoded byte length of every block payload before Base45.
- `compressed_size`: exact byte length of the zlib stream before zero padding.
- `original_size`: exact original file size in bytes.
- `sha256`: full SHA-256 of the original bytes.
- `suffix`: uppercase filename suffix without the dot, or `TXT` as fallback.
- `filename_b32`: UTF-8 basename encoded with RFC 4648 Base32, padding omitted.
- `crc32`: CRC32 of this block's raw `chunk_size` bytes.
- `payload_b45`: RFC 9285 Base45 of those raw block bytes.

The entire QR string stays inside the QR alphanumeric character set, which is materially denser than putting
Base64 in QR byte mode.

## Whole-block recovery code

The `k` source chunks are a byte matrix. VisualMD uses a systematic generator matrix

```text
G = [ I ]
    [ C ]
```

where `I` is the `k x k` identity matrix and `C` is an `r x k` Cauchy matrix over GF(256), using primitive
polynomial `0x11D`.

For parity row `p` and data column `j`:

```text
C[p,j] = inverse_GF256(p XOR (r + j))
```

with `p in [0,r)` and `j in [0,k)`. VisualMD limits `k+r <= 256`, so the x/y sets are distinct. Every square
submatrix required for erasure recovery is nonsingular; consequently any `k` valid blocks out of `k+r` can
recover the source chunks.

This layer is for *whole QR loss*. QR's own ECC M remains responsible for errors inside an individual QR.

## Validation order

A decoder should:

1. Decode QR text.
2. Parse the header and Base45 payload.
3. Require `len(payload) == chunk_size`.
4. Verify that block's CRC32.
5. Reject blocks whose invariant metadata disagrees with the selected file.
6. Once any `k` distinct valid blocks exist, reconstruct the `k` data chunks if necessary.
7. Concatenate, trim to `compressed_size`, and zlib-decompress.
8. Require `len(original) == original_size`.
9. Require `SHA256(original) == sha256`.

Only after step 9 is recovery considered exact.

## Sheet conventions

The reference encoder uses a 4 x 4 grid, QR ECC M, four-module QR quiet zones, and approximately 25% recovery
blocks by default. The block number/type/CRC are printed below every QR, and the complete file SHA-256 is
printed at the top. These human-readable values are redundant; the machine-readable copies inside each QR are
authoritative.

A file may span multiple pages if it does not fit the configured grid. For the intended one-photo workflow,
prefer source files that remain within one page rather than reducing QR modules to an impractically small size.

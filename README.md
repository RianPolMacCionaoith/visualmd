# VisualMD

VisualMD is a camera-friendly way to move Markdown and source-code files through a visual channel.
It turns a file into a sheet of independently decodable QR blocks. You can display the sheet on one
computer, photograph it with a phone, and recover the original bytes from the photo.

The v1 format is optimized for `.md`, `.c`, `.cpp`, `.h`, and `.py`, but it can carry arbitrary files.
Text/source is especially effective because it compresses well.

## What v1 protects against

1. **zlib compression** reduces source/text before visual encoding.
2. **QR ECC level M** handles local blur, scratches, glare, and pixel errors inside each QR.
3. **CRC32 per block** rejects a QR payload unless that whole block is exact.
4. **Cauchy/GF(256) whole-block recovery** creates parity QRs. By default roughly 25% of the block count is redundant. If `r` recovery blocks are generated, *any `r` complete QR blocks may be lost* and the file is still recoverable.
5. **SHA-256 of the original file** is carried in every block and checked after decompression.

The output sheet also prints all block CRC32 values and the file SHA-256 as human-readable text.

## Install

Linux is the primary development target. On Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y python3 python3-venv libzbar0

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e ".[decode]"
```

For encoder-only use, install with `python -m pip install -e .`; ZBar is only needed for photographed-sheet decoding. See [`QUICKSTART_LINUX.md`](QUICKSTART_LINUX.md) for the shortest Linux workflow.

## Encode

```bash
visualmd encode notes.md -o notes_visual.png
visualmd encode main.cpp -o main_visual.png
```

Default layout is 4 x 4 QR positions per page. Default block payload is 1800 raw compressed bytes,
and recovery is about 25%. If the encoded file exceeds the one-sheet capacity, VisualMD automatically
creates multiple `_page_XX.png` files.

Useful controls:

```bash
visualmd encode project_notes.md -o sheet.png --chunk-size 1600 --recovery 3
visualmd encode code.cpp -o sheet.png --columns 4 --rows 4 --box-size 5
```

A smaller chunk makes each QR easier to photograph, at the cost of more QRs. `1800` is an intentionally
conservative starting point for a modern phone camera photographing a high-resolution monitor.

## Decode / inspect a phone photo

```bash
visualmd inspect photo.jpg
visualmd decode photo.jpg -o recovered.md
```

If some blocks are missing, `inspect` shows what was recovered. You can then take a closer photograph
of the unreadable part and pass both photos:

```bash
visualmd decode whole_sheet.jpg closeup.jpg -o recovered.md
```

Duplicate blocks are harmless. Reconstruction only accepts blocks whose CRC32 is correct and finally
verifies the SHA-256 of the exact original file.

## v1 block format

Every QR is an uppercase QR-alphanumeric string:

```text
VMD1:<file-id>:<index>:<k>:<r>:<chunk-size>:<compressed-size>:<original-size>:<sha256>:<suffix>:<Base32 filename>:<crc32>:<Base45 payload>
```

- `k`: number of data blocks
- `r`: number of recovery/parity blocks
- indexes `0..k-1`: data; `k..k+r-1`: parity
- payloads are equal-size bytes before Base45 encoding
- Base45 is used because its alphabet maps efficiently to QR's alphanumeric mode
- `file-id` is the first 12 hex digits of SHA-256
- the original basename is stored as Base32 so recovery can restore a useful filename

The systematic whole-block code is `[I; C]`, where `C` is a Cauchy matrix over GF(256). Therefore any
`k` valid blocks among the `k+r` total blocks can reconstruct all original data blocks.

## Practical single-photo capacity

With the defaults, a one-page 4 x 4 sheet normally holds roughly **12 data QRs + 4 recovery QRs**,
or about **21.6 KB of compressed source data**. Depending on repetition, that can represent perhaps
50-100+ KB of Markdown/C/C++/Python source. Exact capacity depends on compressibility and the chosen
recovery count.

For very large files, use multiple sheets rather than making QR modules too fine for a phone camera.


## Verified reference tests

The v0.1.0 implementation has been exercised end-to-end on a 215,445-byte source-heavy Markdown file:

- compressed to 16,923 bytes; encoded as 10 data + 3 recovery QRs;
- clean generated sheet decoded 13/13 and matched the original byte-for-byte;
- after completely erasing three whole QR blocks and JPEG-compressing the sheet, the remaining 10/13 reconstructed the file and passed SHA-256;
- a synthetic phone-style capture with perspective distortion, uneven illumination, blur, and JPEG compression decoded 13/13 and reconstructed byte-for-byte;
- the GF(256) test suite checks every 5-of-8 recovery combination in a smaller exhaustive code test.

These are reference tests, not a guarantee for every real camera/monitor combination; glare, focus, moire and extreme perspective can still make more blocks unreadable than the configured recovery budget.

## Current limitations

- The decoder prefers ZBar (`pyzbar`) for dense high-version QR codes and keeps OpenCV as a fallback. Real photographs remain camera/lighting dependent; v1 deliberately includes recovery QRs so several codes can be missed.
- Sheet-level corner markers are rendered but v1 does not yet use them for automatic whole-page perspective rectification.
- The basename is stored in every QR as Base32 metadata. Very long filenames increase QR overhead, so v1 is best with ordinary source-file names.
- No encryption. Anyone who can photograph the sheet can decode it.

## Design goal

VisualMD prioritizes **exact recovery and a simple visual handoff**, not steganography or maximum theoretical density.

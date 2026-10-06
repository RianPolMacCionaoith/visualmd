# VisualMD quick start — Linux

VisualMD is developed with Linux as the primary environment.

## 1. Install

On Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y python3 python3-venv libzbar0

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e '.[decode]'
```

For encoder-only use, `libzbar0` and the `decode` extra are not required:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e .
```

## 2. Encode a file

```bash
visualmd encode README.md -o README.visualmd.png
visualmd encode driver.c -o driver.visualmd.png
visualmd encode script.py -o script.visualmd.png
```

Display the PNG large on a monitor and photograph the whole sheet as square-on as practical.

## 3. Inspect or decode a photograph

```bash
visualmd inspect phone_photo.jpg
visualmd decode phone_photo.jpg -o recovered.md
```

If the whole-sheet photo misses some blocks, add close-ups:

```bash
visualmd decode whole_sheet.jpg closeup_1.jpg closeup_2.jpg -o recovered.md
```

Duplicate blocks are harmless. VisualMD accepts only CRC-valid blocks and verifies the reconstructed file with SHA-256.

## 4. Run tests

```bash
python -m pytest -q
```

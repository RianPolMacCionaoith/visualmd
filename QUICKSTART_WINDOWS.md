# VisualMD quick start on Windows

From PowerShell in the extracted `visualmd` folder:

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
py -m pip install -e .
```

Encode a Markdown or source file:

```powershell
py -m visualmd encode C:\path\to\notes.md -o notes_visual.png
py -m visualmd encode C:\path\to\driver.cpp -o driver_visual.png
```

Open the resulting PNG at high resolution/full screen and take a square-on phone photo. For the one-photo workflow,
keep the entire sheet in frame and avoid screen glare. Upload the photo in chat.

If you also want local photo decoding, install the optional decoder dependencies:

```powershell
py -m pip install -e ".[decode]"
py -m visualmd inspect photo.jpg
py -m visualmd decode photo.jpg
```

If PowerShell blocks venv activation, you can skip activation and call the venv Python directly:

```powershell
.venv\Scripts\python.exe -m pip install -e .
.venv\Scripts\python.exe -m visualmd encode notes.md -o notes_visual.png
```

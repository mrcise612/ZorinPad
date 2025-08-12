# ZorinPad

A WordPad-style editor for Zorin OS built with PyQt6. Save and edit rich text as HTML/“.zpad”, with optional export to RTF/DOCX/ODT via Pandoc. Preferences (font, size, wrap, zoom, reopen last file, recent files) persist across sessions.

## ✨ Features

- Rich-text editor (HTML under the hood; native `.zpad`)
- Plain-text mode for `.txt`
- Optional import/export: **RTF/DOCX/ODT** (Pandoc + pypandoc)
- Toolbar: **Bold/Italic/Underline**, color & highlight, align left/center/right/justify
- Lists (bulleted & numbered), **increase/decrease indent**
- Insert/Edit/Open **links**
- **Find/Replace**
- **Word count** + live line/column in status bar
- **Zoom** in/out/reset with persisted steps
- **Word wrap** toggle
- **Preferences** dialog (font, size, wrap, reopen last)
- **Recent Files** menu (10 items) + **Reopen last file on startup**
- **Print** support

## 🧰 Requirements

- Python 3.10+
- PyQt6

Optional (for RTF/DOCX/ODT import/export):
- Pandoc
- pypandoc

### Install on Ubuntu/Zorin
```bash
sudo apt update
sudo apt install -y python3-pyqt6
# Optional export support:
sudo apt install -y pandoc
python3 -m pip install --user pypandoc

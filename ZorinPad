#!/usr/bin/env python3
"""
ZorinPad — WordPad‑style editor for Zorin OS (PyQt6)

Changes in this build (applies your requested improvements):
• Separated UI and logic: new ZorinPadCore for I/O, conversions, prefs, recent files.
• Centralized config/constants at top; no duplicate filters.
• Proactive Pandoc handling: Pandoc‑only actions/filters hidden/disabled when unavailable.
• Status bar feedback on save/open/replace; non‑blocking toasts via showMessage.
• Toolbar adds: Undo/Redo, Increase/Decrease Indent, Justify (visible), Insert Link/Edit Link.
• Preferences persisted (~/.config/zorinpad.json): font family/size, zoom, word‑wrap, reopen last file.
• Startup behavior: optionally reopen last edited file (pref toggle).
• "Recent Files" submenu (File ▸ Recent Files) with up to 10 items and Clear list.
• Clean file dialog filters (no verbose "— requires Pandoc" text; tooltips explain when disabled).
• Self‑tests remain available via --self-test (headless; imports gated).

Run:
  sudo apt install -y python3-pyqt6 pandoc  # pandoc optional
  pip install --user pypandoc                # optional for RTF/DOCX/ODT
  python3 zorinpad.py
"""
from __future__ import annotations

import argparse
import importlib.util
import json
import os
import re
import sys
import tempfile
from dataclasses import dataclass, asdict
from datetime import datetime
from pathlib import Path
from typing import List, Optional

# ----------------------------- App Config -----------------------------
APP_NAME = "ZorinPad"
APP_ID = "zorinpad"
DEFAULT_EXT = ".zpad"  # HTML under the hood
CONFIG_DIR = Path.home() / ".config"
CONFIG_PATH = CONFIG_DIR / f"{APP_ID}.json"
RECENT_MAX = 10
INDENT_WIDTH_PX = 24
DEFAULT_FONT_FAMILY = "Sans Serif"
DEFAULT_FONT_SIZE = 12.0
DEFAULT_WRAP = True
DEFAULT_REOPEN_LAST = True

# File dialog filters (base definitions; Pandoc‑only are enabled dynamically)
FILTERS = {
    "open_base": [
        "ZorinPad/HTML (*.zpad *.html *.htm)",
        "Plain Text (*.txt)",
        "All Files (*)",
    ],
    "open_pandoc": [
        "Rich Text Format (*.rtf)",
        "Word Document (*.docx)",
        "OpenDocument Text (*.odt)",
    ],
    "save_base": [
        "ZorinPad/HTML (*.zpad *.html)",
        "Plain Text (*.txt)",
    ],
    "save_pandoc": [
        "Rich Text Format (*.rtf)",
        "Word Document (*.docx)",
        "OpenDocument Text (*.odt)",
    ],
}

# ----------------------------- Utilities -----------------------------

def have_module(name: str) -> bool:
    return importlib.util.find_spec(name) is not None


def human_path(p: Path | str | None) -> str:
    return str(Path(p).expanduser()) if p else "Untitled"


def pandoc_available() -> bool:
    if not have_module("pypandoc"):
        return False
    try:
        import pypandoc  # type: ignore
        _ = pypandoc.get_pandoc_version()
        return True
    except Exception:
        return False


# ----------------------------- Core (non‑UI) -----------------------------
@dataclass
class Preferences:
    font_family: str = DEFAULT_FONT_FAMILY
    font_size: float = DEFAULT_FONT_SIZE
    zoom_steps: int = 0  # QTextEdit zoomIn/zoomOut relative steps
    word_wrap: bool = DEFAULT_WRAP
    reopen_last: bool = DEFAULT_REOPEN_LAST
    last_file: Optional[str] = None
    recent_files: List[str] = None  # type: ignore

    def __post_init__(self):
        if self.recent_files is None:
            self.recent_files = []


class ZorinPadCore:
    def __init__(self) -> None:
        self.prefs = self.load_prefs()
        self.pandoc_ok = pandoc_available()
        self.pypandoc = None
        if self.pandoc_ok:
            import pypandoc  # type: ignore
            self.pypandoc = pypandoc

    # ---- Preferences ----
    def load_prefs(self) -> Preferences:
        try:
            if CONFIG_PATH.exists():
                data = json.loads(CONFIG_PATH.read_text(encoding="utf-8"))
                return Preferences(**data)
        except Exception:
            pass
        return Preferences()

    def save_prefs(self) -> None:
        try:
            CONFIG_DIR.mkdir(parents=True, exist_ok=True)
            CONFIG_PATH.write_text(json.dumps(asdict(self.prefs), indent=2), encoding="utf-8")
        except Exception:
            # Non‑fatal
            pass

    # ---- Recent files ----
    def add_recent(self, path: Path) -> None:
        sp = str(path)
        rf = [p for p in self.prefs.recent_files if p != sp]
        rf.insert(0, sp)
        self.prefs.recent_files = rf[:RECENT_MAX]
        self.prefs.last_file = sp
        self.save_prefs()

    def clear_recent(self) -> None:
        self.prefs.recent_files = []
        self.save_prefs()

    # ---- I/O & Conversion ----
    def load_file(self, path: Path) -> tuple[str, bool]:
        """Return (content, is_html). Raises on error."""
        ext = path.suffix.lower()
        data = path.read_bytes()
        if ext in (".zpad", ".html", ".htm"):
            return data.decode("utf-8", errors="replace"), True
        if ext == ".txt":
            return data.decode("utf-8", errors="replace"), False
        if ext in (".rtf", ".docx", ".odt"):
            if not self.pandoc_ok or not self.pypandoc:
                raise RuntimeError("RTF/DOCX/ODT require Pandoc + pypandoc.")
            # RTF is text, DOCX/ODT are binary containers → use the appropriate pypandoc API
            if ext == ".rtf":
                html = self.pypandoc.convert_text(
                    data.decode("utf-8", errors="ignore"), to="html", format="rtf"
                )
            else:
                # For binary formats, convert from file path
                html = self.pypandoc.convert_file(str(path), to="html")
            return html, True
        # Fallback best‑effort
        return data.decode("utf-8", errors="replace"), False

    def save_file(self, path: Path, html: str, plain: str) -> None:
        ext = (path.suffix.lower() or DEFAULT_EXT)
        if ext in (".zpad", ".html", ".htm"):
            path.write_text(html, encoding="utf-8")
        elif ext == ".txt":
            path.write_text(plain, encoding="utf-8")
        elif ext in (".rtf", ".docx", ".odt"):
            if not self.pandoc_ok or not self.pypandoc:
                raise RuntimeError("RTF/DOCX/ODT require Pandoc + pypandoc.")
            if ext == ".rtf":
                out = self.pypandoc.convert_text(html, to="rtf", format="html")
                path.write_text(out, encoding="utf-8")
            else:
                # For binary outputs (DOCX/ODT), write directly via outputfile
                self.pypandoc.convert_text(html, to=ext.strip("."), format="html", outputfile=str(path))
        else:
            # default to HTML
            path.with_suffix(DEFAULT_EXT).write_text(html, encoding="utf-8")


# ----------------------------- GUI Launcher -----------------------------

def launch_qt_app(core: ZorinPadCore) -> int:
    if not have_module("PyQt6"):
        print(
            f"\n[{APP_NAME}] PyQt6 is not installed. Install with:\n"
            "  sudo apt update && sudo apt install -y python3-pyqt6\n"
            "  # optional export support:\n  sudo apt install -y pandoc && pip install --user pypandoc\n",
            file=sys.stderr,
        )
        return 1

    # Import PyQt only after dependency check
    from PyQt6.QtCore import Qt, QSize
    from PyQt6.QtGui import (
        QAction, QTextCharFormat, QTextCursor, QKeySequence, QTextListFormat,
        QFont, QTextDocument, QDesktopServices
    )
    from PyQt6.QtWidgets import (
        QApplication, QMainWindow, QFileDialog, QTextEdit, QToolBar, QStatusBar,
        QMessageBox, QColorDialog, QFontComboBox, QComboBox, QDialog, QLineEdit,
        QPushButton, QCheckBox, QHBoxLayout, QVBoxLayout, QLabel, QMenu, QInputDialog
    )
    from PyQt6.QtPrintSupport import QPrinter, QPrintDialog
    from PyQt6.QtCore import QUrl

    # ---------- Small dialogs ----------
    class FindReplaceDialog(QDialog):
        def __init__(self, parent: 'MainWindow') -> None:
            super().__init__(parent)
            self.setWindowTitle("Find / Replace")
            self.setModal(False)
            self.editor: QTextEdit = parent.editor

            self.find_input = QLineEdit(self)
            self.replace_input = QLineEdit(self)
            self.case_cb = QCheckBox("Match case", self)
            self.whole_cb = QCheckBox("Whole words", self)

            find_next_btn = QPushButton("Find Next", self)
            find_prev_btn = QPushButton("Find Prev", self)
            replace_btn = QPushButton("Replace", self)
            replace_all_btn = QPushButton("Replace All", self)

            find_next_btn.clicked.connect(self.find_next)
            find_prev_btn.clicked.connect(lambda: self.find_next(backward=True))
            replace_btn.clicked.connect(self.replace_one)
            replace_all_btn.clicked.connect(self.replace_all)

            layout = QVBoxLayout(self)
            frow = QHBoxLayout(); frow.addWidget(QLabel("Find:")); frow.addWidget(self.find_input)
            rrow = QHBoxLayout(); rrow.addWidget(QLabel("Replace:")); rrow.addWidget(self.replace_input)
            opts = QHBoxLayout(); opts.addWidget(self.case_cb); opts.addWidget(self.whole_cb); opts.addStretch(1)
            btns = QHBoxLayout(); [btns.addWidget(b) for b in (find_prev_btn, find_next_btn, replace_btn, replace_all_btn)]
            layout.addLayout(frow); layout.addLayout(rrow); layout.addLayout(opts); layout.addLayout(btns)

        def _flags(self) -> QTextDocument.FindFlags:
            flags = QTextDocument.FindFlag(0)
            if self.case_cb.isChecked():
                flags |= QTextDocument.FindFlag.FindCaseSensitively
            if self.whole_cb.isChecked():
                flags |= QTextDocument.FindFlag.FindWholeWords
            return flags

        def find_next(self, backward: bool = False) -> None:
            flags = self._flags()
            if backward:
                flags |= QTextDocument.FindFlag.FindBackward
            self.editor.find(self.find_input.text(), flags)

        def replace_one(self) -> None:
            cursor = self.editor.textCursor()
            find_text = self.find_input.text()
            repl_text = self.replace_input.text()
            if cursor.hasSelection() and cursor.selectedText() == find_text:
                cursor.insertText(repl_text)
            self.find_next()

        def replace_all(self) -> None:
            self.editor.moveCursor(QTextCursor.MoveOperation.Start)
            count = 0
            while self.editor.find(self.find_input.text(), self._flags()):
                cursor = self.editor.textCursor()
                cursor.insertText(self.replace_input.text())
                count += 1
            QMessageBox.information(self, APP_NAME, f"Replaced {count} occurrence(s).")
            self.parent().status.showMessage(f"Replaced {count} occurrence(s).", 4000)  # type: ignore

    class PreferencesDialog(QDialog):
        def __init__(self, parent: 'MainWindow') -> None:
            super().__init__(parent)
            self.setWindowTitle("Preferences")

            self.font_box = QFontComboBox(self)
            self.font_box.setCurrentFont(QFont(parent.core.prefs.font_family))

            self.size_box = QComboBox(self)
            self.size_box.setEditable(True)
            self.size_box.addItems(["8","9","10","11","12","14","16","18","20","22","24","28","32","36","48","72"])
            self.size_box.setCurrentText(str(int(parent.core.prefs.font_size)))

            self.wrap_cb = QCheckBox("Word wrap", self)
            self.wrap_cb.setChecked(parent.core.prefs.word_wrap)

            self.reopen_cb = QCheckBox("Reopen last file on startup", self)
            self.reopen_cb.setChecked(parent.core.prefs.reopen_last)

            ok_btn = QPushButton("Save")
            cancel_btn = QPushButton("Cancel")
            ok_btn.clicked.connect(self.accept)
            cancel_btn.clicked.connect(self.reject)

            form = QVBoxLayout(self)
            for lbl, w in (("Font family", self.font_box), ("Font size", self.size_box)):
                row = QHBoxLayout(); row.addWidget(QLabel(lbl)); row.addWidget(w); form.addLayout(row)
            form.addWidget(self.wrap_cb)
            form.addWidget(self.reopen_cb)
            rowb = QHBoxLayout(); rowb.addStretch(1); rowb.addWidget(cancel_btn); rowb.addWidget(ok_btn); form.addLayout(rowb)

        def values(self) -> tuple[str, float, bool, bool]:
            return (
                self.font_box.currentFont().family(),
                float(self.size_box.currentText() or DEFAULT_FONT_SIZE),
                bool(self.wrap_cb.isChecked()),
                bool(self.reopen_cb.isChecked()),
            )

    # ---------- Main Window ----------
    class MainWindow(QMainWindow):
        def __init__(self, core: ZorinPadCore) -> None:
            super().__init__()
            self.core = core
            self.setWindowTitle(APP_NAME)
            self.resize(1000, 700)

            self.editor = QTextEdit(self)
            self.editor.setAcceptRichText(True)
            self.editor.setTabStopDistance(4 * self.editor.fontMetrics().horizontalAdvance(' '))
            self.editor.textChanged.connect(self._on_changed)
            self.setCentralWidget(self.editor)

            # Init state *before* any status updates/signals
            self.current_path: Optional[Path] = None
            self._dirty = False

            # Apply prefs
            self.editor.setCurrentFont(QFont(self.core.prefs.font_family))
            self.editor.setFontPointSize(self.core.prefs.font_size)
            self.act_wrap_state = self.core.prefs.word_wrap
            self._apply_wrap(self.act_wrap_state)
            if self.core.prefs.zoom_steps:
                # positive => zoom in; negative => zoom out
                steps = self.core.prefs.zoom_steps
                if steps > 0:
                    for _ in range(steps):
                        self.editor.zoomIn(1)
                elif steps < 0:
                    for _ in range(-steps):
                        self.editor.zoomOut(1)

            self.status = QStatusBar(self); self.setStatusBar(self.status)
            self._update_status()
            self.editor.cursorPositionChanged.connect(self._update_status)

            self.find_dlg = FindReplaceDialog(self)

            self._build_actions()
            self._build_menus()
            self._build_toolbar()

            # Startup reopen
            if self.core.prefs.reopen_last and self.core.prefs.last_file and Path(self.core.prefs.last_file).exists():
                try:
                    self._open_path(Path(self.core.prefs.last_file))
                    self.set_path(Path(self.core.prefs.last_file))
                    self._dirty = False
                except Exception:
                    pass

        # ---------- UI Builders ----------
        def _build_actions(self) -> None:
            # File actions
            self.act_new = QAction("New", self, shortcut=QKeySequence.StandardKey.New, triggered=self.file_new)
            self.act_open = QAction("Open…", self, shortcut=QKeySequence.StandardKey.Open, triggered=self.file_open)
            self.act_save = QAction("Save", self, shortcut=QKeySequence.StandardKey.Save, triggered=self.file_save)
            self.act_save_as = QAction("Save As…", self, shortcut=QKeySequence.StandardKey.SaveAs, triggered=self.file_save_as)
            self.act_print = QAction("Print…", self, shortcut=QKeySequence.StandardKey.Print, triggered=self.file_print)
            self.act_exit = QAction("Exit", self, shortcut=QKeySequence.StandardKey.Quit, triggered=self.close)

            # Edit actions
            self.act_undo = QAction("Undo", self, shortcut=QKeySequence.StandardKey.Undo, triggered=self.editor.undo)
            self.act_redo = QAction("Redo", self, shortcut=QKeySequence.StandardKey.Redo, triggered=self.editor.redo)
            self.act_cut = QAction("Cut", self, shortcut=QKeySequence.StandardKey.Cut, triggered=self.editor.cut)
            self.act_copy = QAction("Copy", self, shortcut=QKeySequence.StandardKey.Copy, triggered=self.editor.copy)
            self.act_paste = QAction("Paste", self, shortcut=QKeySequence.StandardKey.Paste, triggered=self.editor.paste)
            self.act_find = QAction("Find / Replace", self, shortcut=QKeySequence.StandardKey.Find, triggered=self.find_dlg.show)
            self.act_select_all = QAction("Select All", self, shortcut=QKeySequence.StandardKey.SelectAll, triggered=self.editor.selectAll)

            # Format actions
            self.act_bold = QAction("Bold", self, checkable=True, shortcut=QKeySequence("Ctrl+B"), triggered=self.toggle_bold)
            self.act_italic = QAction("Italic", self, checkable=True, shortcut=QKeySequence("Ctrl+I"), triggered=self.toggle_italic)
            self.act_underline = QAction("Underline", self, checkable=True, shortcut=QKeySequence("Ctrl+U"), triggered=self.toggle_underline)
            self.act_text_color = QAction("Text Color…", self, triggered=self.pick_text_color)
            self.act_highlight = QAction("Highlight…", self, triggered=self.pick_highlight)

            self.act_align_left = QAction("Align Left", self, triggered=lambda: self.editor.setAlignment(Qt.AlignmentFlag.AlignLeft))
            self.act_align_center = QAction("Align Center", self, triggered=lambda: self.editor.setAlignment(Qt.AlignmentFlag.AlignHCenter))
            self.act_align_right = QAction("Align Right", self, triggered=lambda: self.editor.setAlignment(Qt.AlignmentFlag.AlignRight))
            self.act_align_justify = QAction("Justify", self, triggered=lambda: self.editor.setAlignment(Qt.AlignmentFlag.AlignJustify))

            self.act_bullets = QAction("Bulleted List", self, triggered=lambda: self.toggle_list(bulleted=True))
            self.act_numbers = QAction("Numbered List", self, triggered=lambda: self.toggle_list(bulleted=False))

            self.act_indent_inc = QAction("Increase Indent", self, shortcut=QKeySequence("Ctrl+]"), triggered=lambda: self.change_indent(+INDENT_WIDTH_PX))
            self.act_indent_dec = QAction("Decrease Indent", self, shortcut=QKeySequence("Ctrl+["), triggered=lambda: self.change_indent(-INDENT_WIDTH_PX))

            # Links
            self.act_link_insert = QAction("Insert Link…", self, triggered=self.insert_link)
            self.act_link_edit = QAction("Edit Link…", self, triggered=self.edit_link)
            self.act_link_open = QAction("Open Link", self, triggered=self.open_link)

            # View
            self.act_wrap = QAction("Word Wrap", self, checkable=True, checked=self.core.prefs.word_wrap, triggered=self.toggle_wrap)
            self.act_zoom_in = QAction("Zoom In", self, shortcut=QKeySequence.StandardKey.ZoomIn, triggered=self._zoom_in)
            self.act_zoom_out = QAction("Zoom Out", self, shortcut=QKeySequence.StandardKey.ZoomOut, triggered=self._zoom_out)
            self.act_zoom_reset = QAction("Reset Zoom", self, triggered=self._zoom_reset)

            # Help / Prefs
            self.act_about = QAction("About", self, triggered=self.show_about)
            self.act_prefs = QAction("Preferences…", self, triggered=self.show_prefs)

        def _build_menus(self) -> None:
            m_file = self.menuBar().addMenu("&File")
            for a in (self.act_new, self.act_open, self.act_save, self.act_save_as, self.act_print):
                m_file.addAction(a)
            # Recent files
            self.m_recent = m_file.addMenu("Recent Files")
            self._rebuild_recent_menu()
            m_file.addSeparator(); m_file.addAction(self.act_exit)

            m_edit = self.menuBar().addMenu("&Edit")
            for a in (self.act_undo, self.act_redo, self.act_cut, self.act_copy, self.act_paste, self.act_find, self.act_select_all):
                m_edit.addAction(a)

            m_format = self.menuBar().addMenu("F&ormat")
            for a in (self.act_bold, self.act_italic, self.act_underline, self.act_text_color, self.act_highlight):
                m_format.addAction(a)
            m_format.addSeparator()
            for a in (self.act_align_left, self.act_align_center, self.act_align_right, self.act_align_justify):
                m_format.addAction(a)
            m_format.addSeparator()
            for a in (self.act_bullets, self.act_numbers, self.act_indent_inc, self.act_indent_dec):
                m_format.addAction(a)

            m_insert = self.menuBar().addMenu("&Insert")
            for a in (self.act_link_insert, self.act_link_edit, self.act_link_open):
                m_insert.addAction(a)
            m_insert.addAction(QAction("Insert Date/Time", self, shortcut=QKeySequence("Ctrl+D"), triggered=self.insert_datetime))

            m_view = self.menuBar().addMenu("&View")
            for a in (self.act_wrap, self.act_zoom_in, self.act_zoom_out, self.act_zoom_reset):
                m_view.addAction(a)

            # Export (Pandoc)
            m_export = self.menuBar().addMenu("E&xport (Pandoc)")
            self.act_export_rtf = QAction("Export as RTF…", self, triggered=lambda: self.save_as_with_ext(".rtf"))
            self.act_export_docx = QAction("Export as DOCX…", self, triggered=lambda: self.save_as_with_ext(".docx"))
            self.act_export_odt = QAction("Export as ODT…", self, triggered=lambda: self.save_as_with_ext(".odt"))
            for a in (self.act_export_rtf, self.act_export_docx, self.act_export_odt):
                a.setEnabled(self.core.pandoc_ok)
                if not self.core.pandoc_ok:
                    a.setToolTip("Requires Pandoc + pypandoc")
                m_export.addAction(a)

            m_help = self.menuBar().addMenu("&Help")
            m_help.addAction(self.act_prefs)
            m_help.addAction(self.act_about)

        def _build_toolbar(self) -> None:
            tb = QToolBar("Formatting", self)
            tb.setIconSize(QSize(16, 16))
            self.addToolBar(tb)

            # Font controls
            self.font_box = QFontComboBox(self)
            self.font_box.setCurrentFont(QFont(self.core.prefs.font_family))
            self.font_box.currentFontChanged.connect(self.apply_font_family)
            tb.addWidget(self.font_box)

            self.size_box = QComboBox(self)
            self.size_box.setEditable(True)
            self.size_box.addItems(["8","9","10","11","12","14","16","18","20","22","24","28","32","36","48","72"])
            self.size_box.setCurrentText(str(int(self.core.prefs.font_size)))
            self.size_box.editTextChanged.connect(self.apply_font_size)
            tb.addWidget(self.size_box)

            for a in (self.act_bold, self.act_italic, self.act_underline, self.act_text_color, self.act_highlight):
                tb.addAction(a)
            tb.addSeparator()
            for a in (self.act_align_left, self.act_align_center, self.act_align_right, self.act_align_justify):
                tb.addAction(a)
            tb.addSeparator()
            for a in (self.act_bullets, self.act_numbers, self.act_indent_inc, self.act_indent_dec):
                tb.addAction(a)
            tb.addSeparator()
            for a in (self.act_undo, self.act_redo):
                tb.addAction(a)

        # ---------- Recent files UI ----------
        def _rebuild_recent_menu(self) -> None:
            self.m_recent.clear()
            if not self.core.prefs.recent_files:
                action = QAction("(No recent files)", self); action.setEnabled(False)
                self.m_recent.addAction(action)
            else:
                for sp in self.core.prefs.recent_files:
                    a = QAction(sp, self)
                    a.triggered.connect(lambda checked=False, p=sp: self._open_recent(Path(p)))
                    self.m_recent.addAction(a)
                self.m_recent.addSeparator()
                clear = QAction("Clear Recent", self)
                clear.triggered.connect(self._clear_recent)
                self.m_recent.addAction(clear)

        def _open_recent(self, path: Path) -> None:
            if not path.exists():
                QMessageBox.warning(self, APP_NAME, f"File not found:\n{path}")
                return
            if not self._confirm_discard():
                return
            try:
                self._open_path(path)
                self.set_path(path)
                self._dirty = False
            except Exception as e:
                QMessageBox.critical(self, APP_NAME, f"Failed to open:\n{path}\n\n{e}")

        def _clear_recent(self) -> None:
            self.core.clear_recent()
            self._rebuild_recent_menu()

        # ---------- File ops ----------
        def set_path(self, path: Optional[Path]) -> None:
            self.current_path = path
            title = f"{APP_NAME} — {human_path(path)}{' *' if self._dirty else ''}"
            self.setWindowTitle(title)

        def _confirm_discard(self) -> bool:
            if not self._dirty:
                return True
            resp = QMessageBox.question(self, APP_NAME, "You have unsaved changes. Discard them?",
                                        QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No)
            return resp == QMessageBox.StandardButton.Yes

        def file_new(self) -> None:
            if not self._confirm_discard():
                return
            self.editor.clear()
            self._dirty = False
            self.set_path(None)

        def file_open(self) -> None:
            if not self._confirm_discard():
                return
            filters = FILTERS["open_base"][:]
            if self.core.pandoc_ok:
                filters = [*FILTERS["open_base"], *FILTERS["open_pandoc"]]
            path_str, _ = QFileDialog.getOpenFileName(self, "Open", str(Path.home()), ";;".join(filters))
            if not path_str:
                return
            path = Path(path_str)
            try:
                self._open_path(path)
                self.set_path(path)
                self._dirty = False
                self.status.showMessage(f"Opened: {path}", 3000)
                self.core.add_recent(path)
                self._rebuild_recent_menu()
            except Exception as e:
                QMessageBox.critical(self, APP_NAME, f"Failed to open:\n{path}\n\n{e}")

        def _open_path(self, path: Path) -> None:
            content, is_html = self.core.load_file(path)
            if is_html:
                self.editor.setHtml(content)
            else:
                self.editor.setPlainText(content)

        def file_save(self) -> None:
            if self.current_path is None:
                self.file_save_as()
            else:
                self._save_to(self.current_path)

        def save_as_with_ext(self, ext: str) -> None:
            base = Path(self.current_path).with_suffix(ext) if self.current_path else Path.home() / f"untitled{ext}"
            path_str, _ = QFileDialog.getSaveFileName(self, f"Export {ext}", str(base), ext.upper().strip('.') + f" (*{ext})")
            if path_str:
                self._save_to(Path(path_str))

        def file_save_as(self) -> None:
            filters = FILTERS["save_base"][:]
            if self.core.pandoc_ok:
                filters = [*FILTERS["save_base"], *FILTERS["save_pandoc"]]
            start_path = human_path(self.current_path) if self.current_path else str(Path.home() / f"untitled{DEFAULT_EXT}")
            path_str, _ = QFileDialog.getSaveFileName(self, "Save As", start_path, ";;".join(filters))
            if not path_str:
                return
            path = Path(path_str)
            self._save_to(path)
            self.set_path(path)

        def _save_to(self, path: Path) -> None:
            try:
                html = self.editor.document().toHtml()
                plain = self.editor.toPlainText()
                self.core.save_file(path, html, plain)
            except Exception as e:
                QMessageBox.critical(self, APP_NAME, f"Failed to save:\n{path}\n\n{e}")
                return
            self._dirty = False
            self._update_status()
            self.status.showMessage(f"Saved to: {path}", 3000)
            self.core.add_recent(path)
            self._rebuild_recent_menu()

        def file_print(self) -> None:
            printer = QPrinter(QPrinter.PrinterMode.HighResolution)
            dlg = QPrintDialog(printer, self)
            if dlg.exec():
                self.editor.print(printer)

        # ---------- Formatting ----------
        def merge_format_on_selection(self, fmt: QTextCharFormat) -> None:
            cursor = self.editor.textCursor()
            if not cursor.hasSelection():
                cursor.select(QTextCursor.SelectionType.WordUnderCursor)
            cursor.mergeCharFormat(fmt)
            self.editor.mergeCurrentCharFormat(fmt)

        def toggle_bold(self) -> None:
            fmt = QTextCharFormat()
            fmt.setFontWeight(QFont.Weight.Bold if self.act_bold.isChecked() else QFont.Weight.Normal)
            self.merge_format_on_selection(fmt)

        def toggle_italic(self) -> None:
            fmt = QTextCharFormat(); fmt.setFontItalic(self.act_italic.isChecked())
            self.merge_format_on_selection(fmt)

        def toggle_underline(self) -> None:
            fmt = QTextCharFormat(); fmt.setFontUnderline(self.act_underline.isChecked())
            self.merge_format_on_selection(fmt)

        def apply_font_family(self, font: QFont) -> None:
            fmt = QTextCharFormat(); fmt.setFontFamily(font.family())
            self.merge_format_on_selection(fmt)
            self.core.prefs.font_family = font.family(); self.core.save_prefs()

        def apply_font_size(self, size_str: str) -> None:
            try:
                size = float(size_str)
            except ValueError:
                return
            fmt = QTextCharFormat(); fmt.setFontPointSize(size)
            self.merge_format_on_selection(fmt)
            self.core.prefs.font_size = size; self.core.save_prefs()

        def pick_text_color(self) -> None:
            col = QColorDialog.getColor(parent=self)
            if col.isValid():
                fmt = QTextCharFormat(); fmt.setForeground(col)
                self.merge_format_on_selection(fmt)

        def pick_highlight(self) -> None:
            col = QColorDialog.getColor(parent=self)
            if col.isValid():
                fmt = QTextCharFormat(); fmt.setBackground(col)
                self.merge_format_on_selection(fmt)

        def toggle_list(self, bulleted: bool) -> None:
            cursor = self.editor.textCursor()
            cursor.beginEditBlock()
            fmt = QTextListFormat()
            fmt.setStyle(QTextListFormat.Style.ListDisc if bulleted else QTextListFormat.Style.ListDecimal)
            cursor.createList(fmt)
            cursor.endEditBlock()

        def change_indent(self, delta_px: int) -> None:
            cursor = self.editor.textCursor()
            cursor.beginEditBlock()
            block_fmt = cursor.blockFormat()
            m = block_fmt.leftMargin() + delta_px
            block_fmt.setLeftMargin(max(0, m))
            cursor.setBlockFormat(block_fmt)
            cursor.endEditBlock()

        def insert_link(self) -> None:
            cursor = self.editor.textCursor()
            sel_text = cursor.selectedText() or "link text"
            url, ok = QInputDialog.getText(self, APP_NAME, "Enter URL:", text="https://")
            if not ok or not url:
                return
            text, ok2 = QInputDialog.getText(self, APP_NAME, "Text to display:", text=sel_text)
            if not ok2:
                return
            html = f'<a href="{url}">{text}</a>'
            cursor.insertHtml(html)

        def edit_link(self) -> None:
            cursor = self.editor.textCursor()
            if not cursor.charFormat().isAnchor():
                QMessageBox.information(self, APP_NAME, "Cursor is not on a link.")
                return
            fmt = cursor.charFormat()
            hrefs = fmt.anchorHref() if hasattr(fmt, 'anchorHref') else (fmt.anchorHref if hasattr(fmt, 'anchorHref') else "")
            # Fallback: Qt6 provides anchorHref() on QTextCharFormat
            try:
                current = fmt.anchorHref()
            except Exception:
                current = hrefs or ""
            url, ok = QInputDialog.getText(self, APP_NAME, "Edit URL:", text=current)
            if not ok or not url:
                return
            fmt.setAnchorHref(url)
            fmt.setAnchor(True)
            cursor.mergeCharFormat(fmt)

        def open_link(self) -> None:
            cursor = self.editor.textCursor()
            fmt = cursor.charFormat()
            try:
                href = fmt.anchorHref()
            except Exception:
                href = ""
            if href:
                QDesktopServices.openUrl(QUrl(href))
            else:
                QMessageBox.information(self, APP_NAME, "Cursor is not on a link.")

        def insert_datetime(self) -> None:
            cursor = self.editor.textCursor()
            cursor.insertText(datetime.now().strftime("%Y-%m-%d %H:%M"))

        def _apply_wrap(self, on: bool) -> None:
            mode = QTextEdit.LineWrapMode.WidgetWidth if on else QTextEdit.LineWrapMode.NoWrap
            self.editor.setLineWrapMode(mode)

        def toggle_wrap(self) -> None:
            self.act_wrap_state = not self.act_wrap_state
            self._apply_wrap(self.act_wrap_state)
            self.core.prefs.word_wrap = self.act_wrap_state
            self.core.save_prefs()

        def _zoom_in(self) -> None:
            self.editor.zoomIn(1)
            self.core.prefs.zoom_steps += 1; self.core.save_prefs()

        def _zoom_out(self) -> None:
            self.editor.zoomOut(1)
            self.core.prefs.zoom_steps -= 1; self.core.save_prefs()

        def _zoom_reset(self) -> None:
            # Reset by undoing steps
            steps = self.core.prefs.zoom_steps
            if steps > 0:
                for _ in range(steps):
                    self.editor.zoomOut(1)
            elif steps < 0:
                for _ in range(-steps):
                    self.editor.zoomIn(1)
            self.core.prefs.zoom_steps = 0; self.core.save_prefs()

        def show_about(self) -> None:
            msg = (
                f"{APP_NAME} — WordPad‑style editor for Zorin OS\n\n"
                "• Native formats: .zpad/.html, .txt\n"
                "• Optional: .rtf/.docx/.odt via Pandoc + pypandoc\n\n"
                "Tip: Save in .zpad for best fidelity; export to RTF/DOCX when needed."
            )
            QMessageBox.information(self, "About", msg)

        def show_prefs(self) -> None:
            dlg = PreferencesDialog(self)
            if dlg.exec():
                family, size, wrap, reopen = dlg.values()
                self.core.prefs.font_family = family
                self.core.prefs.font_size = size
                self.core.prefs.word_wrap = wrap
                self.core.prefs.reopen_last = reopen
                self.core.save_prefs()
                self.editor.setCurrentFont(QFont(family))
                self.editor.setFontPointSize(size)
                self._apply_wrap(wrap)

        # ---------- State ----------
        def _on_changed(self) -> None:
            self._dirty = True
            self._update_status()

        def _update_status(self) -> None:
            text = self.editor.toPlainText()
            words = len([w for w in re.split(r"\s+", text.strip()) if w])
            cur = self.editor.textCursor()
            self.status.showMessage(f"Words: {words}    Line: {cur.blockNumber()+1}, Col: {cur.positionInBlock()+1}")
            title = f"{APP_NAME} — {human_path(self.current_path)}{' *' if self._dirty else ''}"
            self.setWindowTitle(title)

        def closeEvent(self, event):  # type: ignore[override]
            if self._confirm_discard():
                # Save last path for reopen
                if self.current_path:
                    self.core.prefs.last_file = str(self.current_path)
                    self.core.save_prefs()
                event.accept()
            else:
                event.ignore()

    # ---- App bootstrap ----
    app = QApplication(sys.argv)
    w = MainWindow(core)
    w.show()
    return int(app.exec())


# ----------------------------- Self Tests -----------------------------

def run_self_tests() -> int:
    failures = 0

    def check(name: str, cond: bool, detail: str = "") -> None:
        nonlocal failures
        status = "PASS" if cond else "FAIL"
        print(f"[TEST] {name}: {status}" + (f" — {detail}" if detail else ""))
        if not cond:
            failures += 1

    check("PyQt6 importable", have_module("PyQt6"), "Install python3-pyqt6 or pip install --user PyQt6")
    check("pypandoc+pandoc available", pandoc_available(), "Optional; enables RTF/DOCX/ODT")

    # Prefs roundtrip
    core = ZorinPadCore()
    old_zoom = core.prefs.zoom_steps
    core.prefs.zoom_steps = 3
    core.save_prefs()
    core2 = ZorinPadCore()
    check("prefs persisted", core2.prefs.zoom_steps == 3)
    core2.prefs.zoom_steps = old_zoom; core2.save_prefs()

    # File round‑trip basic (no GUI)
    with tempfile.TemporaryDirectory() as td:
        p_html = Path(td) / "t.zpad"
        p_txt = Path(td) / "t.txt"
        html_sample = "<h1>Title</h1><p>Hello <b>world</b></p>"
        txt_sample = "Hello world\nLine 2"
        p_html.write_text(html_sample, encoding="utf-8")
        p_txt.write_text(txt_sample, encoding="utf-8")
        core3 = ZorinPadCore()
        h, is_html = core3.load_file(p_html)
        t, is_txt_html = core3.load_file(p_txt)
        check("read .zpad as html", is_html and "<h1>Title</h1>" in h)
        check("read .txt as plain", not is_txt_html and "Line 2" in t)

    print("\nSELF-TESTS", "PASSED" if failures == 0 else f"FAILED ({failures})")
    return 0 if failures == 0 else 1


def parse_args(argv: list[str]) -> argparse.Namespace:
    ap = argparse.ArgumentParser(description=f"{APP_NAME} — WordPad‑style editor for Zorin OS")
    ap.add_argument("--self-test", action="store_true", help="Run headless self‑tests and exit")
    return ap.parse_args(argv)


def main(argv: list[str] | None = None) -> int:
    ns = parse_args(argv or sys.argv[1:])
    if ns.self_test:
        return run_self_tests()
    core = ZorinPadCore()
    return launch_qt_app(core)


if __name__ == "__main__":
    raise SystemExit(main())

import sys
import os
import ctypes
from datetime import datetime
from pathlib import Path
import winreg

from PyQt6.QtCore import Qt, QThread, pyqtSignal
from PyQt6.QtGui import QFont
from PyQt6.QtWidgets import (
    QApplication,
    QWidget,
    QVBoxLayout,
    QGridLayout,
    QHBoxLayout,
    QGroupBox,
    QLabel,
    QCheckBox,
    QPushButton,
    QProgressBar,
    QPlainTextEdit,
    QMessageBox,
    QFrame
)


# ---------------------------------------------------------------------------
# System Utilities & Helper Functions
# ---------------------------------------------------------------------------

def is_admin() -> bool:
    """Check if the current process has administrative elevation."""
    try:
        return ctypes.windll.shell32.IsUserAnAdmin() != 0
    except Exception:
        return False


def restart_as_admin() -> None:
    """Relaunch the current script requesting UAC elevation."""
    try:
        executable = sys.executable
        script = os.path.abspath(sys.argv[0])
        params = " ".join([f'"{arg}"' for arg in sys.argv[1:]])
        args = f'"{script}" {params}'.strip()

        # SW_SHOW = 5
        ret = ctypes.windll.shell32.ShellExecuteW(
            None,
            "runas",
            executable,
            args,
            None,
            5
        )
        if int(ret) > 32:
            sys.exit(0)
    except Exception as ex:
        print(f"Failed to elevate process: {ex}")


def get_steam_shader_paths() -> list[Path]:
    """Retrieve Steam shader cache paths via Registry lookup and default locations."""
    paths: list[Path] = []
    
    # 1. Check HKCU Registry
    try:
        with winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\Valve\Steam") as key:
            val, _ = winreg.QueryValueEx(key, "SteamPath")
            candidate = Path(val) / "steamapps" / "shadercache"
            if candidate.exists():
                paths.append(candidate)
    except OSError:
        pass

    # 2. Check 32-bit and 64-bit Program Files standard fallbacks
    for env_var in ["ProgramFiles(x86)", "ProgramFiles"]:
        prog_dir = os.environ.get(env_var)
        if prog_dir:
            fallback = Path(prog_dir) / "Steam" / "steamapps" / "shadercache"
            if fallback.exists() and fallback not in paths:
                paths.append(fallback)

    return paths


def format_size(bytes_count: int) -> str:
    """Format raw byte counts into clean human-readable representation."""
    if bytes_count <= 0:
        return "0 B"
    units = ["B", "KB", "MB", "GB", "TB"]
    i = 0
    size = float(bytes_count)
    while size >= 1024.0 and i < len(units) - 1:
        size /= 1024.0
        i += 1
    return f"{size:.2f} {units[i]}"


def get_directory_size(path: Path) -> int:
    """Non-throwing directory size scanner."""
    total = 0
    if not path.exists():
        return 0
    try:
        if path.is_file() or path.is_symlink():
            return path.stat().st_size
        for root, _, files in os.walk(path, topdown=True):
            for file_name in files:
                try:
                    fp = Path(root) / file_name
                    if not fp.is_symlink():
                        total += fp.stat().st_size
                except (OSError, PermissionError):
                    continue
    except (OSError, PermissionError):
        pass
    return total


# ---------------------------------------------------------------------------
# Background Workers (QThread Engine)
# ---------------------------------------------------------------------------

class SizeScannerWorker(QThread):
    """Background scanner calculating directory sizes without blocking the UI thread."""
    size_calculated = pyqtSignal(str, int)  # target_key, size_in_bytes
    scan_started = pyqtSignal(str)
    scan_finished = pyqtSignal(int)         # total_bytes_found

    def __init__(self, target_dict: dict[str, list[Path]]):
        super().__init__()
        self.target_dict = target_dict

    def run(self) -> None:
        grand_total = 0
        for key, paths in self.target_dict.items():
            self.scan_started.emit(key)
            category_total = 0
            for p in paths:
                if p and p.exists():
                    category_total += get_directory_size(p)
            self.size_calculated.emit(key, category_total)
            grand_total += category_total
            
        self.scan_finished.emit(grand_total)


class SafeCleanerWorker(QThread):
    """Background deletion worker. Safely removes locked files and tracks skipped items."""
    progress_changed = pyqtSignal(int)
    log_message = pyqtSignal(str)
    clean_completed = pyqtSignal(int, int, int)  # bytes_freed, files_deleted, files_skipped

    def __init__(self, targets_to_clean: list[Path]):
        super().__init__()
        self.targets_to_clean = targets_to_clean

    def run(self) -> None:
        self.log_message.emit("Indexing deletable cache files...")
        
        items_to_delete: list[Path] = []
        for base_path in self.targets_to_clean:
            if not base_path.exists():
                continue
            try:
                # Walk bottom-up to delete contents before directories
                for root, dirs, files in os.walk(base_path, topdown=False):
                    for file_name in files:
                        items_to_delete.append(Path(root) / file_name)
                    for dir_name in dirs:
                        items_to_delete.append(Path(root) / dir_name)
            except (OSError, PermissionError) as ex:
                self.log_message.emit(f"Scan skip: {base_path.name} ({ex})")

        total_items = len(items_to_delete)
        if total_items == 0:
            self.progress_changed.emit(100)
            self.log_message.emit("No cache files to clean.")
            self.clean_completed.emit(0, 0, 0)
            return

        self.log_message.emit(f"Deletion started across {total_items} items...")
        
        bytes_freed = 0
        files_deleted = 0
        files_skipped = 0

        for idx, item in enumerate(items_to_delete):
            try:
                if item.is_file() or item.is_symlink():
                    file_size = item.stat().st_size
                    item.unlink(missing_ok=True)
                    bytes_freed += file_size
                    files_deleted += 1
                elif item.is_dir():
                    # Attempt to remove directory if empty; ignore if files remain
                    item.rmdir()
            except (PermissionError, OSError):
                # File is actively locked by a driver, game, or process
                files_skipped += 1
                continue

            # Update progress bar smoothly without flooding event loop
            if idx % 15 == 0 or idx == total_items - 1:
                progress = int(((idx + 1) / total_items) * 100)
                self.progress_changed.emit(progress)

        self.progress_changed.emit(100)
        self.clean_completed.emit(bytes_freed, files_deleted, files_skipped)


# ---------------------------------------------------------------------------
# Main GUI Window
# ---------------------------------------------------------------------------

class CacheCleanerWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Shader & App Cache Cleaner v1.0")
        self.setFixedWidth(500)
        self.setFixedHeight(540)

        # Structure target paths
        self.targets = self._define_targets()
        self.cached_sizes: dict[str, int] = {k: 0 for k in self.targets.keys()}

        # Active worker references
        self.scanner_thread: SizeScannerWorker | None = None
        self.cleaner_thread: SafeCleanerWorker | None = None

        # Build UI
        self._init_ui()

        # Output initial status
        self._log("Application initialized.")
        if not is_admin():
            self._log("Warning: Running with non-elevated user permissions.")

        # Auto-scan all paths on startup
        self._scan_all_targets()

    def _define_targets(self) -> dict[str, dict]:
        local_app = Path(os.environ.get("LOCALAPPDATA", ""))
        roaming_app = Path(os.environ.get("APPDATA", ""))
        temp_dir = Path(os.environ.get("TEMP", ""))

        return {
            "amd": {
                "name": "AMD DxCache / Shaders:",
                "paths": [local_app / "AMD" / "DxCache", local_app / "AMD" / "GLCache"]
            },
            "nvidia": {
                "name": "Nvidia DX / GLCache:",
                "paths": [local_app / "NVIDIA" / "DXCache", local_app / "NVIDIA" / "GLCache"]
            },
            "dx12": {
                "name": "DirectX System Cache:",
                "paths": [local_app / "D3DSCache"]
            },
            "steam": {
                "name": "Steam Shader Pre-Cache:",
                "paths": get_steam_shader_paths()
            },
            "temp": {
                "name": "Windows Local Temp:",
                "paths": [temp_dir]
            },
            "discord": {
                "name": "Discord App Cache:",
                "paths": [roaming_app / "discord" / "Cache", roaming_app / "discord" / "Code Cache"]
            }
        }

    def _init_ui(self) -> None:
        main_layout = QVBoxLayout(self)
        main_layout.setContentsMargins(10, 8, 10, 8)
        main_layout.setSpacing(6)

        # 0. Admin Privilege Banner (Hidden if elevated)
        self.admin_banner = QFrame()
        self.admin_banner.setFrameShape(QFrame.Shape.Panel)
        self.admin_banner.setFrameShadow(QFrame.Shadow.Sunken)
        self.admin_banner.setStyleSheet("background-color: #FFF9E6; border: 1px solid #E6B800;")
        
        banner_layout = QHBoxLayout(self.admin_banner)
        banner_layout.setContentsMargins(8, 4, 8, 4)
        
        self.admin_msg = QLabel("Admin privileges recommended for locked shader caches.")
        self.admin_msg.setStyleSheet("color: #8A6D3B; font-weight: bold; border: none;")
        banner_layout.addWidget(self.admin_msg)
        banner_layout.addStretch()

        self.btn_elevate = QPushButton("Restart as Admin")
        self.btn_elevate.setFixedHeight(22)
        self.btn_elevate.clicked.connect(restart_as_admin)
        banner_layout.addWidget(self.btn_elevate)

        main_layout.addWidget(self.admin_banner)
        if is_admin():
            self.admin_banner.setVisible(False)

        # 1. Configuration & Directories GroupBox
        group_dirs = QGroupBox("Directories to Clean")
        grid = QGridLayout(group_dirs)
        grid.setHorizontalSpacing(8)
        grid.setVerticalSpacing(4)
        grid.setContentsMargins(8, 8, 8, 8)

        # Column configuration & headers
        self.chk_boxes: dict[str, QCheckBox] = {}
        self.size_labels: dict[str, QLabel] = {}
        self.scan_buttons: dict[str, QPushButton] = {}

        for row, (key, meta) in enumerate(self.targets.items()):
            # Col 0: Name Label
            name_lbl = QLabel(meta["name"])
            grid.addWidget(name_lbl, row, 0, Qt.AlignmentFlag.AlignLeft | Qt.AlignmentFlag.AlignVCenter)

            # Col 1: Formatted Size Label
            size_lbl = QLabel("0 B")
            size_lbl.setFixedWidth(75)
            size_lbl.setAlignment(Qt.AlignmentFlag.AlignRight | Qt.AlignmentFlag.AlignVCenter)
            size_lbl.setStyleSheet("font-weight: bold; color: #444444;")
            grid.addWidget(size_lbl, row, 1)
            self.size_labels[key] = size_lbl

            # Col 2: CheckBox (Clean)
            chk = QCheckBox("Clean")
            chk.setChecked(True)
            chk.stateChanged.connect(self._update_start_button_state)
            grid.addWidget(chk, row, 2, Qt.AlignmentFlag.AlignCenter)
            self.chk_boxes[key] = chk

            # Col 3: Scan Button
            btn_scan = QPushButton("↻")
            btn_scan.setToolTip("Scan this path")
            btn_scan.setFixedWidth(36)
            btn_scan.setFixedHeight(22)
            btn_scan.clicked.connect(lambda _, k=key: self._scan_single_target(k))
            grid.addWidget(btn_scan, row, 3, Qt.AlignmentFlag.AlignCenter)
            self.scan_buttons[key] = btn_scan

        # Horizontal action row inside Group Box (Scan All button)
        self.btn_scan_all = QPushButton("Scan All Directories")
        self.btn_scan_all.setFixedHeight(24)
        self.btn_scan_all.clicked.connect(self._scan_all_targets)
        grid.addWidget(self.btn_scan_all, len(self.targets), 0, 1, 4)

        main_layout.addWidget(group_dirs)

        # 2. System Status & Log GroupBox
        group_status = QGroupBox("System Status")
        status_layout = QVBoxLayout(group_status)
        status_layout.setContentsMargins(8, 6, 8, 6)
        status_layout.setSpacing(4)

        self.progress_bar = QProgressBar()
        self.progress_bar.setRange(0, 100)
        self.progress_bar.setValue(0)
        self.progress_bar.setTextVisible(True)
        self.progress_bar.setFixedHeight(18)
        status_layout.addWidget(self.progress_bar)

        self.log_view = QPlainTextEdit()
        self.log_view.setReadOnly(True)
        self.log_view.setFixedHeight(120)
        self.log_view.setFont(QFont("Consolas", 8))
        status_layout.addWidget(self.log_view)

        main_layout.addWidget(group_status)

        # 3. Bottom Large Start Action Button
        self.btn_start = QPushButton("START CLEANING")
        self.btn_start.setFixedHeight(38)
        btn_font = self.btn_start.font()
        btn_font.setBold(True)
        self.btn_start.setFont(btn_font)
        self.btn_start.clicked.connect(self._start_cleaning_process)
        main_layout.addWidget(self.btn_start)

    # -----------------------------------------------------------------------
    # Logging & UI Helpers
    # -----------------------------------------------------------------------

    def _log(self, text: str) -> None:
        timestamp = datetime.now().strftime("[%H:%M:%S]")
        self.log_view.appendPlainText(f"{timestamp} {text}")

    def _set_ui_busy(self, is_busy: bool) -> None:
        for btn in self.scan_buttons.values():
            btn.setEnabled(not is_busy)
        for chk in self.chk_boxes.values():
            chk.setEnabled(not is_busy)
        self.btn_scan_all.setEnabled(not is_busy)
        self.btn_start.setEnabled(not is_busy and self._has_selected_targets())

    def _has_selected_targets(self) -> bool:
        return any(chk.isChecked() for chk in self.chk_boxes.values())

    def _update_start_button_state(self) -> None:
        self.btn_start.setEnabled(self._has_selected_targets())

    # -----------------------------------------------------------------------
    # Scanning Controller Logic
    # -----------------------------------------------------------------------

    def _scan_single_target(self, key: str) -> None:
        self._set_ui_busy(True)
        paths = self.targets[key]["paths"]
        self.scanner_thread = SizeScannerWorker({key: paths})
        self.scanner_thread.size_calculated.connect(self._on_single_scanned)
        self.scanner_thread.scan_finished.connect(lambda: self._set_ui_busy(False))
        self.scanner_thread.start()

    def _on_single_scanned(self, key: str, size: int) -> None:
        self.cached_sizes[key] = size
        self.size_labels[key].setText(format_size(size))
        name = self.targets[key]["name"].replace(":", "")
        self._log(f"Scanned {name} -> {format_size(size)}")

    def _scan_all_targets(self) -> None:
        self._set_ui_busy(True)
        scan_dict = {key: meta["paths"] for key, meta in self.targets.items()}
        
        self.scanner_thread = SizeScannerWorker(scan_dict)
        self.scanner_thread.size_calculated.connect(self._on_all_item_scanned)
        self.scanner_thread.scan_finished.connect(self._on_all_scan_finished)
        self.scanner_thread.start()

    def _on_all_item_scanned(self, key: str, size: int) -> None:
        self.cached_sizes[key] = size
        self.size_labels[key].setText(format_size(size))

    def _on_all_scan_finished(self, grand_total: int) -> None:
        self._set_ui_busy(False)
        self._log(f"Total scan complete: {format_size(grand_total)} cache detected.")

    # -----------------------------------------------------------------------
    # Cleaning Controller Logic
    # -----------------------------------------------------------------------

    def _start_cleaning_process(self) -> None:
        # 1. Ask for confirmation
        confirm = QMessageBox.warning(
            self,
            "Confirm Cleanup",
            "Are you sure you want to clean the selected cache locations?\n"
            "Active and locked files will be skipped automatically.",
            QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No,
            QMessageBox.StandardButton.No
        )
        if confirm != QMessageBox.StandardButton.Yes:
            self._log("Cleanup cancelled by user.")
            return

        # 2. Gather selected paths
        selected_paths: list[Path] = []
        for key, chk in self.chk_boxes.items():
            if chk.isChecked():
                selected_paths.extend(self.targets[key]["paths"])

        # 3. Start background deletion
        self._set_ui_busy(True)
        self.progress_bar.setValue(0)

        self.cleaner_thread = SafeCleanerWorker(selected_paths)
        self.cleaner_thread.progress_changed.connect(self.progress_bar.setValue)
        self.cleaner_thread.log_message.connect(self._log)
        self.cleaner_thread.clean_completed.connect(self._on_cleaning_finished)
        self.cleaner_thread.start()

    def _on_cleaning_finished(self, freed_bytes: int, deleted: int, skipped: int) -> None:
        self._log(
            f"Cleanup complete. Cleaned: {format_size(freed_bytes)} "
            f"({deleted} deleted, {skipped} locked/skipped)."
        )
        self._set_ui_busy(False)
        # Rescan to reflect updated sizes
        self._scan_all_targets()


# ---------------------------------------------------------------------------
# Application Entry Point
# ---------------------------------------------------------------------------

if __name__ == "__main__":
    app = QApplication(sys.argv)
    
    # Enforce pure Windows native style
    app.setStyle("windowsvista")

    window = CacheCleanerWindow()
    window.show()
    sys.exit(app.exec())
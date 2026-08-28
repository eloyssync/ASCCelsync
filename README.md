# ASCCelsync (Shader & App Cache Cleaner)

ASCCelsync is a high-performance Windows desktop utility designed for automated discovery, analysis, and safe sanitization of graphics shader caches, runtime system artifacts, and temporary application directories. Built with Python and PyQt6, the application provides a native, low-overhead interface for direct storage optimization without registry bloat or background telemetry.

> **Note on Antivirus Detections:**
> Standalone Windows executables built with packaging tools may trigger 2-3 false positive detections on some antivirus engines. 
> The project is 100% open source. Check our [VirusTotal Report](https://www.virustotal.com/gui/file/e5f3d2fc8bc419399edf185128b7b1b6b391443ae0a401424a4e6e9243a2b0e9?nocache=1).

---
<img width="499" height="568" alt="image" src="https://github.com/user-attachments/assets/58859318-ab46-4d88-84e2-b3689ec4badc" />

## Key Features

- **Automated Directory Discovery:** Scans default local paths for AMD DxCache/GLCache, Nvidia DXCache/GLCache, DirectX D3DSCache, Steam Shader Pre-Cache, Windows Local Temp, and Discord cache repositories.
- **Multithreaded Storage Engine:** Background calculation and deletion operations powered by non-blocking worker threads (`QThread`), preventing GUI lockups during I/O spikes.
- **Safe File Deletion Pipeline:** Implements granular error handling for in-use and process-locked files, safely bypassing active handles and preventing operational crashes.
- **Elevation Management:** Integrated Windows UAC detection with direct elevation escalation triggers for accessing protected system caches.
- **Real-Time Diagnostics:** Monospaced event logging terminal displaying exact byte counts, processed objects, and skipped file metrics.
- **Native GUI Architecture:** Lightweight Windows Vista/10/11 visual design layout optimized for standard system DPI scaling.

---

## Target Directories

| Target Category | Default Path Environment |
| :--- | :--- |
| AMD Shader Cache | `%LOCALAPPDATA%\AMD\DxCache`, `%LOCALAPPDATA%\AMD\GLCache` |
| Nvidia Cache | `%LOCALAPPDATA%\NVIDIA\DXCache`, `%LOCALAPPDATA%\NVIDIA\GLCache` |
| DirectX System Cache | `%LOCALAPPDATA%\D3DSCache` |
| Steam Shader Pre-Cache | `%PROGRAMFILES(X86)%\Steam\steamapps\shadercache` |
| Windows Temp | `%TEMP%` |
| Discord Cache | `%APPDATA%\discord\Cache` |

---

## Installation & Requirements

[Download ASCCelsync v2.0.0](https://github.com/eloyssync/ASCCelsync/releases/tag/v1.0.0)

### Prerequisites

- Windows 10 / Windows 11 (x64)
- Python 3.10+ (for source builds)
- Administrator privileges (recommended for protected DirectX and driver directories)

### Clone & Setup

```bash
git clone https://github.com/eloyssync/ASCCelsync.git
cd ASCCelsync
Install Dependencies
Bash
pip install -r requirements.txt
(Requirements: PyQt6)

Usage
Run from Source
Bash
python src/main.py
Build Standalone Executable
To compile into a self-contained, console-free Windows executable using PyInstaller:

Bash
pyinstaller --noconfirm --onefile --windowed --name "ASCCelsync" src/main.py
The resulting binary will be located inside the dist/ directory.

Operational Workflow
Initialization: The utility evaluates system permission levels and runs an initial scan across all predefined targets.

Selective Sanitization: Check or uncheck target caches via the interface grid according to your cleaning requirements.

Execution: Click START CLEANING to begin multithreaded deletion. Active file locks are automatically recorded in the diagnostic log.

Verification: Post-cleanup re-scanning calculates remaining bytes and displays total reclaimed disk capacity.

Security & Reliability
Non-Destructive Target Scope: Operations are strictly limited to caching directories and temporary file structures. No registry modifications or system core libraries are altered.

Exception Isolation: Individual I/O operations are wrapped with PermissionError and OSError handlers to guarantee stability during active program executions.

License
This project is licensed under the MIT License. See the LICENSE file for complete details.

Developed by eloyssync.

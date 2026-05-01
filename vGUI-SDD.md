# Software Design Document (SDD) - GUI Version
**Project Name:** Nomor SEP Renamer Automation (Pro Desktop Edition)  
**Version:** 1.0.0  
**Date:** May 2026  

---

## 1. Project Overview
The **Nomor SEP Renamer Automation (Pro Desktop Edition)** is a production-ready standalone Windows application designed to provide end-users with a seamless, visual experience for processing PDF documents and ZIP archives. 

Built on top of the robust `BatchProcessor` core, this GUI edition utilizes the modern `CustomTkinter` framework. It features dynamic layout toggling (Stacked vs Side-by-Side), live Dark/Light theme switching, an animated progress tracker, and a customized premium terminal interface for real-time extraction monitoring. It is designed to run entirely locally and air-gapped, requiring no external dependencies or technical knowledge from the end-user.

---

## 2. Architecture Diagrams

```mermaid
graph TD
    User([End User]) --> MainApp[CustomTkinter App <br> Main Thread]
    
    MainApp --> Layout[Layout Manager <br> Stacked / Side-by-Side]
    MainApp --> Theme[Theme Manager <br> Dark / Light]
    MainApp --> InputValid[Directory / File Selection]
    
    InputValid -- "Start Processing" --> ThreadDispatcher[Thread Dispatcher]
    
    subcase Background Processing
        ThreadDispatcher --> WorkerThread[Daemon Worker Thread]
        WorkerThread --> Preprocess{Source Type?}
        Preprocess -- "ZIP Archive" --> Unzip[Extract to Temp]
        Preprocess -- "Multi-page PDF" --> Split[Split Pages to Temp]
        Preprocess -- "Directory" --> Map[Map Directory]
        
        Unzip --> CoreProcessor
        Split --> CoreProcessor
        Map --> CoreProcessor[BatchProcessor Engine]
    end
    
    CoreProcessor -- "Emits Logs" --> PythonLogging[Standard Python Logger]
    PythonLogging --> CustomHandler[TextboxHandler]
    
    CustomHandler -- "self.after()" --> TerminalUI[Premium UI Terminal]
    CoreProcessor -- "Finish/Exception" --> AlertManager[MessageBox Alerts]
    AlertManager -- "self.after()" --> MainApp
```

---

## 3. Module Specifications

### `gui_app.py` (Presentation Layer)
The presentation layer is entirely encapsulated within `gui_app.py`. It is responsible for bridging the user's intent to the underlying data processors safely without locking the operating system UI.

- **`App` (Main Class):** Inherits from `ctk.CTk`. Constructs the entire layout, binds input variables, and manages the main event loop.
- **`TextboxHandler` (Custom Log Router):** Inherits from `logging.Handler`. It intercepts pure Python logs emitted by the background threads and safely pushes them to the CustomTkinter UI using thread-safe `.after()` callbacks, applying syntax highlighting (Green for INFO, Red for ERROR, etc.).
- **Thread Management:** The `_process_thread()` function isolates the heavy I/O operations (PDF text extraction and zip unzipping) onto a background Daemon thread, preventing the "Application Not Responding" freeze in Windows.

---

## 4. Interface Definitions

### 4.1 Internal API & Callbacks
- `_toggle_theme(self) -> None`: Intercepts the user's theme preference and swaps the global `ctk` appearance mode. Simultaneously adjusts the background color of the terminal container to maintain high contrast.
- `_toggle_layout(self, layout_mode: str) -> None`: Re-packs the UI frames dynamically. Adjusts the window geometry to `850x780` for Stacked mode, or `1100x600` for Side-by-Side mode.
- `_run_processing(self) -> None`: Locks the UI inputs, initializes the indeterminate progress bar, clears the previous terminal output, and sparks the Daemon Worker Thread.
- `TextboxHandler.emit(self, record) -> None`: Formats the log `record` and routes it to `self.textbox.after(0, self.append_text, msg, record.levelname)`.

### 4.2 UI State Representation
The application state is bound directly to Tkinter variables to allow passive real-time UI updates:
- `source_var`: `ctk.StringVar()` - Holds absolute path of input target.
- `target_var`: `ctk.StringVar()` - Holds absolute path of output target.
- `run_mode_var`: `ctk.StringVar()` - Toggles between `"Dry Run (Simulation)"` and `"Execute"`.
- `file_mode_var`: `ctk.StringVar()` - Toggles between `"Copy Files"` and `"Move Files"`.

---

## 5. Data Models

The GUI constructs an overriding data payload before sending it to the core engine.

| Setting Component | Type | User Action Mapping | Description |
|-------------------|------|---------------------|-------------|
| `config.source_dir` | `str` | Browse Folder / File | Re-mapped to a temporary directory if a ZIP/PDF file is provided. |
| `config.target_dir` | `str` | Browse Folder | Final resting place for renamed files. |
| `config.dry_run` | `bool` | Execution Segmented Button | Evaluates `True` if "Dry Run (Simulation)" is selected. |
| `config.move_on_success`| `bool` | File Handling Segmented Button | Evaluates `True` if "Move Files" is selected. |

---

## 6. Security & Thread Safety Requirements

1. **Thread-Safe UI Updates:** Tkinter is famously non-thread-safe. Any background thread attempting to modify UI components will cause a `RuntimeError`. The system strictly utilizes `self.after(0, lambda: ...)` to push UI updates (like MessageBoxes and Terminal lines) back onto the main loop queue safely.
2. **Resource Isolation & Cleanup:** When the user uploads a `.zip` or multi-page `.pdf`, the GUI silently provisions an isolated `tempfile.mkdtemp()` environment. To prevent disk bloat, a hard `finally:` block is guaranteed to execute, purging the temporary staging ground immediately after the BPJS renaming finishes.
3. **Air-gapped Guarantee:** Operates strictly on local file systems without requiring any network permissions.

---

## 7. Performance Criteria

1. **Non-Blocking Interface:** The user interface must maintain a 60fps refresh rate (allowing the progress bar to animate smoothly and the terminal to scroll) even while the CPU is completely saturated extracting 50 PDFs concurrently in the background.
2. **Visual Overflow Safety:** The terminal widget is hard-capped on font size and wrapped at boundaries to prevent buffer overflows from extreme logging activity (e.g., parsing thousands of pages).

---

## 8. Deployment Guidelines

The GUI is explicitly designed to be distributed as a standalone desktop application.

### 8.1 PyInstaller Compilation
The app utilizes `build_exe.py` to freeze the Python environment, bundling `CustomTkinter`'s themes, fonts, and core engines (`PyPDF2`, `pdfplumber`) into a single output:
```powershell
python build_exe.py
```
This yields `Nomor_SEP_Renamer.exe` in the `/dist` directory.

### 8.2 Windows Inno Setup (MSI/Setup Installer)
To provide users with a standard enterprise installation experience:
1. Execute `installer_config.iss` using Inno Setup.
2. This produces a `Setup.exe` that automatically installs the program to `C:\Program Files`, creates a Start Menu entry, and places the executable shortcut on the Desktop.
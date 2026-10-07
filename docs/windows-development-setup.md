# Native Windows development setup

Assessed September 22, 2026, against this checkout.

This checklist prepares Windows to build Reco, calibrate ScoutCam recordings,
and produce stitched videos using the RTX 5060 Ti. WSL is optional. Run the
commands below in **Windows PowerShell**, preferably Visual Studio's **Developer
PowerShell configured for x64**, rather than an Ubuntu/WSL terminal.

Repository location: `C:\Users\Rjgoo\Developer\video-stitcher`.

## Required installations

Install these in order. All native tools and libraries should target **Windows
x64**, with the Rust **MSVC** toolchain.

| Done | Install | Required selection / purpose |
| --- | --- | --- |
| [ ] | Microsoft Visual Studio Build Tools | Select **Desktop development with C++**, the MSVC x64/x86 compiler tools, and a Windows SDK. Supplies the compiler, linker, and system headers. |
| [ ] | Git for Windows | Make Git available from PowerShell. Cargo fetches a Git dependency as well as crates.io packages. |
| [ ] | Rustup | Install Rust **1.92.0**, Cargo, rustfmt, and Clippy for `x86_64-pc-windows-msvc`. |
| [ ] | LLVM / Clang | Install the x64 Windows distribution including **libclang.dll**. Used to generate FFmpeg and OBS bindings. |
| [ ] | FFmpeg shared development bundle | Prefer the repo's Windows CI baseline: **FFmpeg 7.1, win64, GPL shared**. Must include headers, link libraries, DLLs, `ffmpeg.exe`, and `ffprobe.exe`. |
| [ ] | Environment variables | Set `FFMPEG_DIR`, `LIBCLANG_PATH`, and the PATH entries described below. |
| [ ] | GPU verification | Test the existing NVIDIA Windows driver with Reco. No driver reinstall is currently justified by the assessment. |

### 1. Microsoft C++ build tools

Download [Build Tools for Visual Studio](https://visualstudio.microsoft.com/visual-cpp-build-tools/).
In Visual Studio Installer, select **Desktop development with C++** and confirm
the MSVC x64/x86 tools and Windows SDK are selected. A full Visual Studio IDE is
not required. Follow [Microsoft's Rust setup guidance](https://learn.microsoft.com/en-us/windows/dev-environment/rust/setup).

After installation, open the installed Developer PowerShell and select its x64
development environment. Verify that `Get-Command cl, link` resolves the Microsoft
compiler and linker. An ordinary PowerShell may not have these on PATH even
though the tools are installed.

### 2. Git for Windows

Use the [Git for Windows installer](https://git-scm.com/install/windows), or:

```powershell
winget install --id Git.Git -e --source winget
```

Git was found in WSL during assessment, but not on the checked Windows PATH.
Reopen the terminal after installing.

### 3. Rust

Download and run the Windows installer from [Install Rust](https://rust-lang.org/tools/install/).
Choose the default MSVC toolchain, then reopen PowerShell and run:

```powershell
rustup toolchain install 1.92.0 --profile minimal --component rustfmt --component clippy
Set-Location 'C:\Users\Rjgoo\Developer\video-stitcher'
rustup show active-toolchain
rustc --version
cargo --version
```

The repo's `rust-toolchain.toml` selects 1.92.0 automatically inside this directory.
The active toolchain should be `1.92.0-x86_64-pc-windows-msvc`. Do not change the
repository pin simply because Rustup also installs a newer stable version.

### 4. LLVM / Clang

Use the Windows x64 installer linked from [LLVM releases](https://releases.llvm.org/).
The expected default location is `C:\Program Files\LLVM`. Select the installer
option to add LLVM to PATH if offered.

Check that `C:\Program Files\LLVM\bin\libclang.dll` exists. The Clang executable
alone is insufficient: bindgen loads the library. See the
[bindgen Windows requirements](https://rust-lang.github.io/rust-bindgen/requirements.html).

### 5. FFmpeg

The repository's Windows workflow selects a BtbN archive matching
`*n7.1*win64-gpl-shared*.zip`. Start with that version to match the repo's build
recipe. [BtbN FFmpeg releases](https://github.com/BtbN/FFmpeg-Builds/releases)
currently advertise newer series on the latest release page; use the release
history to locate 7.1. If a matching archive is unavailable, selecting and testing
a newer supported FFmpeg version is a separate compatibility step. Do not assume
the newest download matches this checkout.

Extract the bundle to a stable directory, for example `C:\Tools\ffmpeg-7.1`,
with this structure directly beneath it:

```text
C:\Tools\ffmpeg-7.1\
    bin\       ffmpeg.exe, ffprobe.exe, FFmpeg DLLs
    include\   libavcodec, libavformat, libavutil, etc.
    lib\       link libraries
```

An executable-only or static command-line bundle does not supply the complete
shared development setup. Keep the headers, libraries, and DLLs from the same
archive. The Rust dependency named `ffmpeg-next = "8"` is a crate version; it
does not supersede the Windows CI recipe's FFmpeg 7.1 selection.

### 6. Environment variables

Open **Edit environment variables for your account** in Windows. Add:

| User variable | Example value |
| --- | --- |
| `FFMPEG_DIR` | `C:\Tools\ffmpeg-7.1` |
| `LIBCLANG_PATH` | `C:\Program Files\LLVM\bin` |

Add these entries to your existing **user Path**, preserving its other entries:

```text
C:\Tools\ffmpeg-7.1\bin
C:\Program Files\LLVM\bin
%USERPROFILE%\.cargo\bin
```

Adjust the paths to match your actual installation. `FFMPEG_DIR` must point to
the directory containing `bin`, `include`, and `lib`, not to `bin` itself.
Reopen terminals and editors after changing user variables.

For a temporary setup in the current PowerShell session only:

```powershell
$env:FFMPEG_DIR = 'C:\Tools\ffmpeg-7.1'
$env:LIBCLANG_PATH = 'C:\Program Files\LLVM\bin'
$env:Path = "$env:FFMPEG_DIR\bin;$env:LIBCLANG_PATH;$env:USERPROFILE\.cargo\bin;$env:Path"
```

### 7. NVIDIA graphics

The assessment detected an RTX 5060 Ti with approximately 16 GB VRAM and an
installed NVIDIA Windows driver. Reco selects **DirectX 12 by default on Windows**.
Windows and the graphics driver provide the graphics runtime; a standalone Vulkan
SDK or CUDA Toolkit is not required for the initial calibration workflow.

The WSL Vulkan probe exposed only the CPU renderer `llvmpipe`; this is why native
Windows is the preferred first execution environment. Native Reco GPU execution
still needs to be verified after building.

## Verify installation and build the initial CLI

Run in Windows Developer PowerShell with the x64 compiler environment:

```powershell
Set-Location 'C:\Users\Rjgoo\Developer\video-stitcher'
Get-Command git, rustup, rustc, cargo, clang, ffmpeg, ffprobe
Get-Command cl, link
git --version
rustc --version
cargo fmt --version
cargo clippy --version
clang --version
ffmpeg -version
ffprobe -version
Test-Path "$env:LIBCLANG_PATH\libclang.dll"
Test-Path "$env:FFMPEG_DIR\include\libavcodec\avcodec.h"
Get-ChildItem "$env:FFMPEG_DIR\lib"
```

Build the CLI without AI features for the ScoutCam calibration task:

```powershell
cargo build --locked --release -p reco-cli --no-default-features
```

Stop and resolve any build failure before running the next commands; PowerShell
does not necessarily stop automatically when a native command fails.

```powershell
.\target\release\reco.exe --help
.\target\release\reco.exe info
.\target\release\reco.exe calibrate --help
.\target\release\reco.exe stitch --help
```

Confirm `info` reports the NVIDIA adapter. Device enumeration is the first gate;
a short rendered clip must still demonstrate that the real pipeline works.
Follow `prompts/calibrate-scoutcam-20260920-173128.md` for footage verification,
lens profiles, synchronization, and rendering. Preserve the original videos.

Initial CLI checks:

```powershell
cargo test --locked -p reco-cli --no-default-features
cargo clippy --locked -p reco-cli --all-targets --no-default-features -- -D warnings
cargo clippy --locked -p reco-cli --all-targets --no-default-features --features profiling -- -D warnings
cargo fmt --all -- --check
```

These targeted checks do not replace the repository's complete workspace CI.
The initial CLI build avoids the optional AI runtime and OBS development setup.
Network access is needed to fetch Cargo dependencies, including a Git dependency.

## Optional installations by task

| Task | Additional installation or configuration |
| --- | --- |
| Edit Rust comfortably | [VS Code](https://code.visualstudio.com/) with the rust-analyzer extension; optionally CodeLLDB for debugging. Use a native Windows window for this setup. |
| Work with GitHub issues and PRs | [GitHub CLI](https://cli.github.com/); authenticate when needed. |
| Run the desktop GUI with no AI | No separate Slint SDK required; Cargo builds Slint. Start with `cargo build --locked --release -p reco-gui --no-default-features`, then run `target\release\reco-gui.exe`. |
| Enable AI tracking | A compatible ONNX Runtime and a suitable YOLO model. Default builds use the Cargo ORT dependency; builds with `load-dynamic` require runtime DLLs supplied separately. |
| Match the Windows GUI CI runtime bundle | The checked-in workflow uses `load-dynamic`, ONNX Runtime DirectML **1.24.4**, and DirectML **1.15.4**. Follow that workflow's DLL extraction and bundling recipe; these are repository baseline versions, not a claim about the latest releases. |
| Enable CUDA inference | Compatible CUDA/cuDNN runtime libraries for the selected ONNX Runtime build and `cuda` feature. Choose versions together when implementing this backend. |
| Work on TensorRT or NCNN | Backend SDK/libraries and possibly CMake/Ninja for source builds. The native backend build scripts contain Linux-oriented paths and linking assumptions; a Windows port may require code work as well as installation. |
| Test GStreamer integration | [GStreamer for Windows](https://gstreamer.freedesktop.org/documentation/installing/on-windows.html): matching x64 MSVC runtime and development packages, plus pkg-config discovery configuration. Required only for the `gstreamer` feature. Linux/Jetson camera sources still need their target platform. |
| Develop the OBS plugin / build the full workspace | [OBS Studio](https://obsproject.com/) for execution, plus matching OBS development headers and Windows import libraries. Set `OBS_INCLUDE_DIR` to the directory containing `obs.h`. Installing the OBS application alone is insufficient. |
| Reproduce extra security checks | `cargo-deny`, `cargo-audit`, and Gitleaks, matching repository CI. Install compatible tool versions; these developer tools may require a newer toolchain than the project's pinned compiler. |
| Build cloud worker containers later | Docker Desktop is already present, but its engine and integration were not verified. Linux container work may use Docker's WSL2 backend even when local Reco runs natively on Windows. |
| Access AWS from Windows later | AWS CLI v2 for Windows. The confirmed AWS CLI installation was in WSL; it does not establish a native Windows installation. |
| Run Python helper scripts | Native Windows Python and the packages required by the particular script. Python is not required to build the Rust CLI. |

**Full-workspace caveat:** `cargo build`, `cargo test --all`, and workspace Clippy
include `reco-obs`. Its build script assumes parts of the Linux OBS layout and
linking model. Supplying headers is necessary but does not establish that the
Windows plugin will link successfully. The Windows CLI/GUI workflows build those
packages individually. Qualify the OBS build separately before claiming all
workspace checks pass on Windows; document actual API friction in the relevant
crate's `FRICTION.md` as instructed by the repository.

## Not required for the initial calibration task

- WSL or Ubuntu packages.
- CUDA Toolkit, cuDNN, TensorRT, NCNN, or JetPack.
- OpenCV, PyTorch, or a Python machine-learning environment.
- Node.js dependencies or an npm install.
- GStreamer or OBS development libraries for the targeted CLI build.
- A standalone Slint installation or Vulkan SDK.
- AWS provisioning, Docker containers, or new GPU hardware.

## Repository references and assessment limits

- [Pinned Rust toolchain](../rust-toolchain.toml)
- [Build prerequisites](../README.md)
- [Windows GUI build and runtime bundling](../.github/workflows/test-build-gui.yml)
- [Release builds](../.github/workflows/release.yml)
- [CLI features](../crates/reco-cli/Cargo.toml)
- [GUI features](../crates/reco-gui/Cargo.toml)
- [GPU backend selection](../crates/reco-core/src/gpu/mod.rs)
- [OBS build requirements](../crates/reco-obs/build.rs)
- [ScoutCam calibration handoff](../prompts/calibrate-scoutcam-20260920-173128.md)

This document is an installation plan based on source inspection and read-only
machine checks. No software was installed, and the Windows build and execution
commands above have not yet been run. Tools missing from the checked PATH may
exist elsewhere on disk; verify before installing duplicates.

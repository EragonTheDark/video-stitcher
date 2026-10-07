# Reco Video Stitcher

Open-source GPU-accelerated panoramic sports camera software.

## Project status

Active release work lives in GitHub issues, PRs, and milestones, not in this
file - check there for the current focus. For the latest stable version and
changelog, see the [GitHub Releases page](https://github.com/reco-project/video-stitcher/releases).
Per-crate consumer pain is logged in each crate's `FRICTION.md`
(e.g. `crates/reco-gui/FRICTION.md`, `crates/reco-obs/FRICTION.md`).

**Rule: document friction, don't work around it.** A reco-core API gap that a
consumer would otherwise hack around gets a `FRICTION.md` entry, not a
consumer-side workaround.

## Architecture (Rust + wgpu)

- `crates/reco-core/` — GPU stitching engine (library crate, no I/O deps)
- `crates/reco-cli/` — CLI binary (`reco stitch`, `reco info`, `reco calibrate`, `reco preview`)
- `crates/reco-io/` — Pluggable I/O backends (FFmpeg decode/encode, GStreamer, libcamera)
- `crates/reco-detect/` — AI detection backends (ORT CPU/GPU, TensorRT, NCNN, CoreML/Metal)
- `crates/reco-autocam/` — AI camera control (directors, trajectory smoothing, ROI filtering)
- `crates/reco-calibrate/` — Stereo camera calibration (AKAZE features, optimization)
- `crates/reco-gui/` — Slint GUI consumer (wgpu zero-copy preview)
- `crates/reco-obs/` — OBS Studio source plugin (async-frame ingestion + BGRA + interactive pan/zoom)

## Key commands

```bash
cargo build                   # Build all crates
cargo test --all              # Run all tests
cargo clippy --all-targets -- -D warnings   # Lint
cargo fmt --all -- --check    # Format check
cargo fmt --all               # Auto-format
cargo doc --no-deps --open    # Generate and open docs
cargo run -p reco-cli -- info # Show GPU info
cargo run -p reco-cli -- stitch left.mp4 right.mp4 -c match.json -o out.mp4
cargo run -p reco-cli -- preview left.mp4 right.mp4 -c match.json
cargo run --release -p reco-cli --features profiling -- stitch left.mp4 right.mp4 -c match.json -o out.mp4 --max-frames 300  # Profile 300 frames → reco-trace.json (open in ui.perfetto.dev)
```

## Build & contributing

- Build prerequisites (Rust version, FFmpeg development libraries, clang,
  pkg-config) and the feature-flag matrix: see [README.md](README.md).
- Contribution conventions (branch naming, PR template, CLA): see
  [CONTRIBUTING.md](CONTRIBUTING.md).

## Local machine setup (Rjgoo — verified September 22, 2026)

- Prefer **native Windows** for local builds, calibration, and stitching. The
  user uses **Visual Studio Developer PowerShell with the x64 environment**.
  WSL is available but is not required for this workflow. Do not infer that
  Windows tools are missing merely because they are absent from WSL's PATH.
- Windows checkout: `C:\Users\Rjgoo\Developer\video-stitcher`.
  The same checkout is visible in WSL at
  `/mnt/c/Users/Rjgoo/Developer/video-stitcher`; it does not need to be moved.
- GPU: **NVIDIA GeForce RTX 5060 Ti, approximately 16 GB VRAM**, with an
  installed Windows NVIDIA driver. Reco defaults to **DirectX 12 on Windows**.
  CUDA Toolkit is not required for the initial non-AI calibration/stitch task.
- FFmpeg installation: **8.1 shared Windows x64 bundle** at
  `C:\Tools\ffmpeg-8.1`; user `FFMPEG_DIR` points there and user PATH contains
  `C:\Tools\ffmpeg-8.1\bin`. Matching avcodec-62, avformat-62, avutil-60,
  swscale-9, and other FFmpeg DLLs were found there. Use this installation;
  the older 7.1 reference in CI/setup docs is not a local version requirement.
- Windows user PATH includes `C:\Users\Rjgoo\.cargo\bin` and
  `C:\Program Files\LLVM\bin`. Use the repository-pinned Rust **1.92.0**
  MSVC toolchain (`x86_64-pc-windows-msvc`). Recheck tool versions or
  `LIBCLANG_PATH` if diagnosing a build; their current values were not captured
  in the successful executable-help verification.
- Existing binary: `target\release\reco.exe`. Both `calibrate --help` and
  `stitch --help` printed their help and exited **0**. This confirms startup
  and argument parsing, not GPU execution, codec operation, or a successful
  rendered stitch. Those runtime checks remain to be performed.
- WSL: Ubuntu 24.04 on WSL2. NVIDIA is visible to `nvidia-smi`, but the Vulkan
  probe exposed only CPU `llvmpipe`. Do not equate CUDA visibility with a
  working hardware Vulkan path or redirect this task to WSL GPU processing.
- Prefer a targeted CLI build for the current calibration work:
  `cargo build --locked --release -p reco-cli --no-default-features`.
  Whole-workspace commands also include OBS and require its development setup;
  that Windows setup has not been verified. Preserve the project's full CI
  requirements when contributing code.

### Running the Windows binary from a WSL-hosted agent

Use `powershell.exe -NoProfile` to execute Windows PowerShell code. This session
initially inherited a stale PATH; simple invocation returned no captured output
and an unset `$LASTEXITCODE`. Do not treat an empty result or the PowerShell
wrapper's exit code as proof that Reco succeeded. The following process-local
PATH refresh and explicit process capture successfully verified both commands:

```powershell
$env:Path = [Environment]::GetEnvironmentVariable('Path', 'Machine') + ';' +
    [Environment]::GetEnvironmentVariable('Path', 'User')
Set-Location 'C:\Users\Rjgoo\Developer\video-stitcher'
foreach ($verb in @('calibrate', 'stitch')) {
    $stdoutFile = [IO.Path]::GetTempFileName()
    $stderrFile = [IO.Path]::GetTempFileName()
    try {
        $process = Start-Process -FilePath '.\target\release\reco.exe' `
            -ArgumentList @($verb, '--help') -NoNewWindow -Wait -PassThru `
            -RedirectStandardOutput $stdoutFile -RedirectStandardError $stderrFile
        Write-Output "$verb exit code: $($process.ExitCode)"
        Get-Content $stdoutFile
        Get-Content $stderrFile
        if ($process.ExitCode -ne 0) { throw "$verb failed: $($process.ExitCode)" }
    } finally {
        Remove-Item $stdoutFile, $stderrFile
    }
}
```

For builds, initialize the installed Visual Studio x64 developer environment;
refreshing PATH alone is not a substitute for its compiler/SDK variables.
Calibration audio sync is disabled with **`--auto-sync false`**; the help text's
mention of `--no-auto-sync` disagrees with the actual listed argument.
See [Windows setup checklist](docs/windows-development-setup.md) and
[ScoutCam calibration handoff](prompts/calibrate-scoutcam-20260920-173128.md)
for further context. Installation checklist items are plans unless separately
verified; do not reinstall software or change drivers based on stale checklist
status.

### ScoutCam calibration results — September 22, 2026

See the [ScoutCam 720p30 calibration report](calibration-results/scoutcam-20260920-173128/report.md)
for measured lens profiles, fitting diagnostics, exact commands, and three
rendered validation clips. The calibration is **provisional**, not production-validated:
black render borders, stretched outer edges, and synchronization/coverage checks
remain unresolved. Artifacts include `match-provisional.json` and per-camera
`*-lens-provisional.json` in the same results directory. Original recordings
remain unchanged. Results apply to the September 20 720p session; do not blindly
reuse profiles for 1080p. Generated artifacts are local and uncommitted.

## Code standards

- `rustfmt` formatting (config in `rustfmt.toml`)
- `clippy` linting with `-D warnings` (zero warnings policy)
- Doc comments (`///`) on all public items
- Module-level docs (`//!`) explaining purpose
- Tests in each module (`#[cfg(test)] mod tests`)
- All PRs must pass: `cargo fmt --check && cargo clippy && cargo test`
- Clippy must also pass with `--features profiling`
- Keep commit messages and PR descriptions concise and technical (what
  changed + why), especially when written by an AI agent - no filler, no
  marketing tone
- `profiling` feature: opt-in `tracing` + `tracing-chrome` instrumentation (zero-cost when off)

## Context
- Public open-source project (AGPL-3.0) with a growing community and forum
- Users include football clubs, amateur sports teams — prioritize UX clarity
- Open alternative to proprietary sports camera solutions
- Targets: desktop (Win/macOS/Linux), NVIDIA Jetson, cloud, mobile

## When writing code
- Production-grade: handle errors, validate inputs at API boundaries
- Cross-platform (Windows/macOS/Linux) — avoid platform-specific assumptions
- Performance matters: this processes video frames in real time
- Modular: reco-core must be usable as a standalone Rust crate
- Explicit over implicit: no hidden defaults, no magic
- Verify changes actually run - exercise the binary or tests, not just `cargo check`

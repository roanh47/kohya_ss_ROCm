# kohya_ss

## Purpose

Gradio-based GUI and CLI front end for [kohya-ss/sd-scripts](https://github.com/kohya-ss/sd-scripts), used to configure and launch Stable Diffusion / SDXL / FLUX training runs (LoRA, DreamBooth, fine-tuning, TI) without hand-writing command lines.

## Ownership

- Maintainer: bmaltais
- This fork (`roanh47/kohya_ss_ROCm`) targets **Windows + AMD Radeon** and deliberately stays one commit away from upstream: the intended divergence is `requirements_pytorch_windows.txt` (plus the fork notes in this file). `upstream` is wired to `bmaltais/kohya_ss`, so keep merges trivial and don't refactor upstream code just to host the fork.
- `sd-scripts/` is a **git submodule** pointing at upstream kohya-ss/sd-scripts. Never edit files inside it — GUI-side changes only. If a fix requires a sd-scripts change, describe it as a snippet in the PR body instead of committing to the submodule.

## Local Contracts

- Python 3.10–3.11 (`pyproject.toml`), dependency management via `uv` (never bare `pip`).
- Fork exception: on Windows the supported install is the pip route — `setup.bat` → `setup/setup_windows.py` → `requirements_pytorch_windows.txt`. The `uv` route pins CUDA wheels (see Fork Notes below), so `uv` is not the installer for this fork's AMD target.
- `kohya_gui.py` is the launcher entry point; it shares a name with the `kohya_gui/` package, so anything loading it dynamically (see `test/test_allowed_paths.py`) must import by file path, not `import kohya_gui`.
- Config precedence: `config.toml` (user, gitignored) vs `config example.toml` (tracked template) — copy, don't edit the example in place.
- Secrets (API tokens, HF keys) go through env vars / `config.toml`, never hardcoded.

## Work Guidance

- Black formatting on Python changes before commit.
- Keep GUI-only changes decoupled from submodule internals — call `sd-scripts` via its existing CLI/library surface, don't reach into its internals from `kohya_gui/`.
- Before a refactor touching >300 LOC, strip dead props/exports/debug logging first, as its own commit.

## Fork Notes: Windows ROCm (AMD)

This fork exists to run kohya_ss on **Windows + AMD Radeon (gfx120X, RDNA3/RDNA4)**. Everything below applies to that target only; the upstream prerequisites (NVIDIA GPU, CUDA 12.8 Toolkit) do not.

- Install: `git clone --recursive` → `setup.bat` (pip route). `requirements_pytorch_windows.txt` is the only changed file: it takes `torch`/`torchvision` from `https://rocm.nightlies.amd.com/v2/gfx120X-all/` and pulls the `rocm` / `rocm-sdk-*` runtime wheels in as dependencies — no CUDA toolkit, no separate ROCm/HIP install, no extra scripts or files.
- Start: `gui.bat` (or `python kohya_gui.py`). `gui.bat`'s requirements validation passes against the ROCm file — don't reach for `--noverify` to paper over it.
- The `uv` route (`gui-uv.bat`) is **not** ROCm-aware: `pyproject.toml` / `uv.lock` pin `torch==2.7.0+cu128` from `download.pytorch.org` and never read `requirements_pytorch_windows.txt`. Send AMD users to the pip route; making the lock ROCm-aware is the work item if uv support is ever wanted.
- Don't add `xformers`, `onnxruntime-gpu`-style CUDA extras: there are no Windows ROCm builds and sd-scripts runs on SDPA (`sdpa = true`).
- `--use-rocm` is inert on Windows; only `gui.sh` / `setup.sh --use-rocm` act on it (Linux).
- Verified on a fresh clone (2026-10-05): `setup.bat --headless` → `torch 2.13.0a0+rocm7.13.0a20260416`, `torch.version.hip 7.2.0`, `torch.cuda.is_available()` True, `AMD Radeon RX 9070 XT`; `gui.bat --headless` serves HTTP 200.
- A `Could not detect ROCm GPU architecture` / `ROCm GPU architecture detection failed` line printed when `bitsandbytes` imports is harmless on this target — the GUI still starts.

## Verification

- `.github/workflows/typos.yaml` runs `crate-ci/typos` on push/PR — fix flagged typos before merging.
- `tests/` holds the pytest regression suite; `test/test_allowed_paths.py` is a standalone unittest. Run both before shipping GUI changes (`pytest tests/ test/test_allowed_paths.py`).
- No `tsc`/`eslint` equivalent exists (pure Python project) — rely on the tests above plus manual GUI smoke-test via `gui.sh` / `gui.bat`.

## Child DOX Index

- `kohya_gui/AGENTS.md` — Gradio GUI package (tabs, shared widget classes, launcher helpers)
- `tools/AGENTS.md` — standalone CLI utility scripts (extraction, conversion, captioning)
- `docs/AGENTS.md` — feature guides and localized documentation
- `tests/AGENTS.md` — pytest regression suite
- `test/AGENTS.md` — manual end-to-end scratch fixtures and the allowed-paths unittest
- `sd-scripts/` — upstream submodule, out of scope for local AGENTS.md (do not create one; see Ownership above)

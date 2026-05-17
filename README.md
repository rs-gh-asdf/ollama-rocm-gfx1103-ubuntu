# ollama-rocm-gfx1103-ubuntu

Native ROCm acceleration for [Ollama](https://github.com/ollama/ollama) on the AMD **Radeon 780M iGPU** (`gfx1103`, the integrated GPU in Ryzen 7040 / 8040 / Phoenix series), on Ubuntu Linux.

This repo is a one-command setup that automates a process documented in detail in [this blog post](https://ataary.com/ubuntu-linux-with-ollama-rocm-on-amd-ryzen-780m-igpu/). If you've got a Phoenix iGPU and want to run local LLMs on it without falling back to CPU, this is for you.

## What it does

Native ROCm `gfx1103` support on Linux is missing from every standard place — Ubuntu's `librocblas5`, AMD's official ROCm .debs, and Ollama's bundled libraries all ship kernels for surrounding architectures (gfx1030/1100/1101/1102/1151/1200) but skip Phoenix. The result is that Ollama enumerates the iGPU as a ROCm device, then crashes when it tries to actually run a GEMM operation, and silently falls back to CPU.

The setup script in this repo closes the gap by:

1. Building [`likelovewant/ollama-for-amd`](https://github.com/likelovewant/ollama-for-amd) from source with `-DAMDGPU_TARGETS=gfx1103`.
2. Applying three small patches to `ml/device.go` — see [`patches/`](patches/).
3. Downloading Fedora 43's `rocblas-6.4.0-7.fc43.x86_64.rpm` and extracting just the `gfx1103` Tensile kernel files (the actual GPU bytecode) into the system rocBLAS library directory. The kernel binaries are distribution-agnostic at the binary level — Fedora's compile, Ubuntu's runtime, no problem.
4. Writing a systemd drop-in so the resulting setup survives reboot and (optionally) pins Ollama to a specific GPU on multi-GPU systems.

After running, Ollama reports `library=ROCm compute=gfx1103` and inference runs at ~24–48 tokens/sec on `gemma4:e2b` (vs ~16 tok/s CPU-only).

## Requirements

- Ubuntu 26.04 LTS (other Ubuntu/Debian versions probably work but untested)
- AMD Phoenix-family CPU (Ryzen 7040 / 8040 / etc.) with Radeon 780M iGPU
- ROCm 7.1 system packages from Ubuntu's universe repo (the script installs these)
- A few GB of disk for the build artifacts and Fedora RPM
- 10–30 minutes of CPU time for the C++/HIP build

If you've got a *different* AMD GPU (gfx1100 / gfx1101 / gfx1102 / gfx1151), this isn't strictly necessary for you — Ubuntu/AMD probably ship working kernels for your arch. Use this only if you actually need gfx1103.

## Quick start

```bash
git clone https://github.com/johnsonfarmsus/ollama-rocm-gfx1103-ubuntu.git
cd ollama-rocm-gfx1103-ubuntu

# Single-GPU system (only the Phoenix iGPU):
./setup.sh

# Multi-GPU system — pin to the iGPU's physical HIP index
# (check `rocminfo` to find which index is the 780M; typically 1 if you also have a dGPU):
ROCR_PIN_DEVICE=1 ./setup.sh
```

The script is idempotent — re-running is safe. In particular, **re-run after every `apt upgrade librocblas5`**, because the upgrade overwrites `/usr/lib/x86_64-linux-gnu/rocblas/5.1.0/library/` with Ubuntu's gfx1103-less package contents.

## Verification

After the script finishes, check that Ollama actually sees the iGPU as ROCm:

```bash
sudo journalctl -u ollama -n 30 | grep "inference compute"
```

You want to see `library=ROCm compute=gfx1103`, not `library=cpu`. If you see CPU, something failed silently — read the surrounding journal output for the actual error.

Then pull a model and inference:

```bash
ollama pull gemma4:e2b   # or any model you like
curl -s http://127.0.0.1:11434/api/chat -d '{
  "model": "gemma4:e2b",
  "stream": false,
  "messages": [{"role": "user", "content": "What is the capital of France?"}]
}' | python3 -m json.tool
```

You should get an answer in a few seconds. Compute `eval_count / eval_duration * 1e9` from the response for tokens/sec.

## What's in here

- [`setup.sh`](setup.sh) — the automation script. Idempotent.
- [`patches/`](patches/) — three patches to apply against `ml/device.go` in `ollama-for-amd`:
  - [`01-sort-by-free-memory.patch`](patches/01-sort-by-free-memory.patch) — fixes the device-selection sort so the iGPU isn't downranked relative to a smaller dGPU. This is upstreamable as a real correctness fix.
  - [`02-skip-init-validation.patch`](patches/02-skip-init-validation.patch) — skips Ollama's "deep init" GPU validation that would crash even working iGPUs.
  - [`03-respect-parent-rocr-visible-devices.patch`](patches/03-respect-parent-rocr-visible-devices.patch) — keeps the runner subprocess on the GPU pinned via systemd, even after Ollama's internal scheduler runs.
- [`override.conf`](override.conf) — systemd drop-in template installed at `/etc/systemd/system/ollama.service.d/override.conf`.

## What's not in here (and why)

- **Patched `rocBLAS` source.** Building rocBLAS from source with `-DAMDGPU_TARGETS=gfx1103` is an alternative to grabbing Fedora's prebuilt kernels. It takes hours and produces functionally the same result. The script uses Fedora's binaries because they work, they're already built, and the .hsaco files are pure GPU bytecode that doesn't care about your host distribution.
- **A binary release.** The `ollama` binary and `libggml-hip.so` need to be built on a machine with ROCm 7.x dev headers matching your runtime. A pre-built binary would have to match a specific runtime version; instead, the script rebuilds each time you run it.

## Caveats / shelf life

This whole approach is held together by a specific set of gaps in the ecosystem. Some of them will close in time:

- When Ubuntu eventually ships gfx1103 kernels in their `librocblas5` package (likely in 27.04 or whenever ROCm 7.x integrates Phoenix), step 6 of the script becomes unnecessary.
- When upstream Ollama integrates the AMD-tuned code (or fixes the sort bias separately), the patches in this repo may stop applying cleanly. If `git apply --check` fails, the patches need to be re-derived against the new upstream.
- An `apt upgrade` of `librocblas5` will overwrite the kernel files. Re-run `setup.sh` afterward.

None of these are imminent as of May 2026.

## Contributing

If you find the patches don't apply against a newer ollama-for-amd checkout, or the Fedora RPM URL changes, open an issue or PR. The script tries to be defensive (refuses to proceed if it can't find expected files) but the moving parts mean breakage is inevitable eventually.

The first patch — sort by free memory — has been submitted upstream to Ollama proper. Track at [#TODO-pr-link].

## Sources & references

- [ROCm/rocBLAS issue #1536 — gfx1103 (Phoenix) support request](https://github.com/ROCm/rocBLAS/issues/1536) (still open as of May 2026)
- [Fedora rocblas package](https://packages.fedoraproject.org/pkgs/rocblas/) — where the gfx1103 kernels come from
- [likelovewant/ollama-for-amd](https://github.com/likelovewant/ollama-for-amd) — the Ollama fork with broader AMD GPU support
- [likelovewant/ROCmLibs-for-gfx1103-AMD780M-APU](https://github.com/likelovewant/ROCmLibs-for-gfx1103-AMD780M-APU) — Windows-focused but useful background
- [Blog post writeup](https://ataary.com/ubuntu-linux-with-ollama-rocm-on-amd-ryzen-780m-igpu/) — the full story of why each piece is necessary

## License

The patches and setup script in this repo are released under the MIT license; see [LICENSE](LICENSE). The upstream projects (Ollama, ollama-for-amd, rocBLAS, Fedora rocblas package) retain their own licenses.

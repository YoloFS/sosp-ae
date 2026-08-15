# YoloFS — SOSP 2026 Artifact

This is the artifact for the SOSP 2026 paper:

> **Don't Let AI Agents YOLO Your Files: Shifting Information and Control to
> Filesystems for Agent Safety and Autonomy**
> Shawn (Wanxiang) Zhong, Junxuan Liao, et al. — University of Wisconsin–Madison
> arXiv: [2604.13536](https://arxiv.org/abs/2604.13536) · Project: <https://yolofs.github.io/>

YoloFS (`\yolofs`) is a stackable Linux filesystem that gives coding agents a
safe place to work: it **stages** every mutation for review, supports
continuous **snapshots** and time-travel, and **gates** file access with
per-path permission rules.

## What this artifact covers

This artifact reproduces the **performance evaluation (§6)** of the paper:

| Paper item | Claim | Reproduced by |
|---|---|---|
| Table (Single-file I/O) | YoloFS adds ~no overhead to 1 GB fio read/write; OverlayFS/BranchFS add overhead | `perf-eval` fio benchmarks |
| Figure (Metadata operations) | YoloFS ≤ OverlayFS on metadata; faster than Ext4 for internal readdir/rename/unlink; BranchFS >20× slower; gating overhead negligible (≤4% for stat) | `perf-eval` metadata benchmarks |
| Figure (Snapshot scalability) | YoloFS stays flat as snapshots grow; OverlayFS fails at ~50 snapshots; BranchFS degrades | `perf-eval` checkpoint-scaling |
| Figure (Realistic workload) | YoloFS ≈ Ext4 on a kernel-dev workload (+3.5 s to commit 100k files); OverlayFS 18% slower | `perf-eval` dev-workflow (macro) |

**Target badges:** Artifacts Available, Functional, and Results Reproduced.

## ⚠️ Warning: this loads a kernel module

`filesystem/` builds and loads a **Linux kernel module** (`insmod`/`rmmod`,
requires `root`). A buggy or interrupted run can wedge mounts or require a
reboot. **Do not run this on a machine you care about.** Use the CloudLab
machine we provide (below), or a disposable VM.

## Hardware access — we provide a CloudLab machine

The paper's numbers come from **CloudLab** `c6525-25g`: AMD EPYC 7302P (16-core,
3 GHz), 128 GB DDR4-3200, Ubuntu 24.04, Linux 6.8, Ext4 on a SATA SSD.
Performance is hardware-sensitive, so **we will provide each reviewer with
access to a matching CloudLab machine** — please request access through the
HotCRP artifact submission and we will provision one. The machine
comes with the OS, kernel headers, and toolchains ready.

The artifact also runs on other recent Linux hosts (see dependencies), but exact
numbers require the matching hardware; the qualitative trends hold broadly.

## Repository layout

This is a thin superproject; the code and data live in submodules pinned to the
evaluated commits:

```
sosp-ae/
├── filesystem/     # YoloFS kernel module (C) + CLI (Rust), build system, tests
├── perf-eval/      # benchmark harness (Rust) + run/report scripts
└── perf-results/   # reference results from the paper's run + HTML dashboard
```

## Getting the artifact

The archival copy (with a DOI) is on Zenodo (see the artifact submission). To
work from source, clone with submodules:

```bash
git clone --recurse-submodules https://github.com/YoloFS/sosp-ae.git
cd sosp-ae
# if you forgot --recurse-submodules:
git submodule update --init --recursive
```

## Dependencies

On the provided CloudLab machine these are preinstalled. On your own Ubuntu
24.04 / Linux 6.8 host:

```bash
sudo apt-get update
sudo apt-get install -y build-essential linux-headers-$(uname -r) fio git python3 libcap2-bin
# Rust toolchain (stable)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
# uv (Python env/plotting for the report figures)
curl -LsSf https://astral.sh/uv/install.sh | sh
```

`perf-eval/setup.sh` installs the generic benchmark dependencies (fio, perf
tools, bpftrace). The BranchFS baseline is built and installed separately with
`perf-eval/scripts/setup_branchfs.sh` (see Full reproduction below); OverlayFS
is part of the stock kernel and needs no extra setup.

## Kick-the-tires (~5 min)

Confirm the module builds, loads, and a benchmark runs end to end:

```bash
# 1. build + install the kernel module and the `yolo` CLI
cd filesystem
make install
```
After this, you should have `yolo` in your path and `yolo --help` prints usage.


```bash
# 2. run tests
make test
```

We also run `make test` in CI on every push, so you can confirm the
expected result without a machine:
<https://github.com/YoloFS/filesystem/actions>.

```bash
# 3. build the benchmark harness and run one microbenchmark (1 iteration)
cd perf-eval
cargo build --release
sudo ./target/release/yolo-bench --workload write-files --backend yolo-no-perm --runs 1
```

You should see a line like `iter 1/1 … NNN ms (init … + stage … + commit …)` and
an HTML report written under `perf-results/report/`.

## Full reproduction

Each experiment is a single script under `perf-eval/`. They handle building,
installing, loading, and unloading the module around the run. Expect the full
suite to take on the order of **1–2 hours** (fio and the macro workload
dominate). See `perf-eval/README.md` for the complete workload/backend matrix
and per-experiment resource notes.

```bash
cd perf-eval
./setup.sh                 # one-time: generic deps (fio, perf tools, bpftrace)
./scripts/setup_branchfs.sh   # one-time: build + install the BranchFS baseline
                              # (needed for the BranchFS bars in every figure/table)

./micro.sh            # microbenchmarks: write/overwrite/rename/unlink + snapshot-scalability
./macro.sh            # macrobenchmark: the realistic kernel-dev workload
```

Results accumulate in `perf-results/report/results.json` (each run merges in,
preserving other workloads/backends).

### Turning results into paper figures/tables

```bash
cd perf-eval
./report.sh                # HTML dashboard → perf-results/report/index.html (open in a browser)
./paper.sh                 # paper-style artifacts → paper/generated/
                           #   fio.tex (Table), metadata.pdf, checkpoint.pdf, dev.pdf (+ .csv data)
```

`report.sh` produces a self-contained HTML dashboard with one page per workload
(plots + tables) — the easiest way to eyeball results against the paper. On the
CloudLab machine (no local browser), serve it over HTTP and open it from your
laptop (forward the port with `ssh -L 8000:localhost:8000 <cloudlab-host>`):

```bash
python -m http.server -d ~/sosp-ae/perf-results/report/   # then browse http://localhost:8000/
```

`paper.sh` regenerates the exact figures/tables used in the paper into a local
`paper/generated/` directory.

**Prefer not to run anything first?** The paper's reference run is published as a
rendered dashboard you can browse directly:
<https://yolofs.github.io/perf-results/report/>. It's the same output `report.sh`
generates, from the `perf-results` committed in this artifact.

### Comparing to the paper

- **Single-file I/O table**: compare `paper/generated/fio.tex` (or the fio pages
  in the HTML dashboard) to the paper's I/O throughput table — YoloFS should be
  within noise of Base (Ext4); OverlayFS/BranchFS show the reported slowdowns.
- **Metadata figure**: `metadata.pdf` — YoloFS ≤ OverlayFS across ops; BranchFS
  far slower; the `yolo-realistic` (gating on) vs `yolo-no-perm` bars show the
  negligible gating overhead.
- **Snapshot scalability**: `checkpoint.pdf` — YoloFS flat vs OverlayFS/BranchFS
  rising; OverlayFS stops at ~50 snapshots.
- **Realistic workload**: `dev.pdf` — YoloFS ≈ Ext4; OverlayFS ~18% slower.

Absolute numbers depend on hardware; on the provided CloudLab machine they should
match the paper closely. On other hardware, expect the same relative ordering
and trends.

## License

This artifact is distributed under the GNU General Public License v2.0 (see
`LICENSE`), matching the YoloFS kernel module. It permits comparison and
extension.

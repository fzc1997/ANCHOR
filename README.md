# ANCHOR

ANCHOR: **A**llelic i**N**ference in single-**C**ell **H**aplotype-aware l**O**ng-**R**eads — de novo,
haplotype- and isoform-resolved allelic expression from single-cell long-read RNA sequencing.

Version: 1.0.0 2026/06/08

Author: Zhican Fu (correspondence: see the manuscript)

> **Source code is provided as a password-protected archive for confidential peer review.**
> The ANCHOR software tool (the three engines + bundled trained models) is distributed in this
> repository as **`ANCHOR-source-encrypted-v1.0.0.zip`**. The decompression **password is provided in
> the manuscript** (Code availability section). The source will be released openly, unencrypted, upon
> publication.
>
> The code that reproduces every figure and benchmark in the paper is in the companion repository
> **[ANCHOR-paper](https://github.com/fzc1997/ANCHOR-paper)** (same password).
>
> Archive SHA-256: `32f8c63146ab7b0aac620c616679080211a282dc8cc8d199a10984ff1cb5a173`


## 1. Installation

### 1.1 Download the ANCHOR repository and decompress the source archive with the password

```bash
git clone https://github.com/fzc1997/ANCHOR.git
cd ANCHOR
unzip ANCHOR-source-encrypted-v1.0.0.zip      # enter the manuscript password → extracts anchor/
```

(You may also simply download `ANCHOR-source-encrypted-v1.0.0.zip` from the repository page instead of
`git clone`. The archive opens with any standard tool — Windows Explorer, macOS double-click, `unzip`,
7-Zip — by entering the manuscript password.)

After extraction:

```
anchor/              # the ANCHOR software tool (3 engines + bundled trained models)
```

### 1.2 Create the conda environment and activate

```bash
cd anchor
conda env create -f environment.yml
conda activate anchor
pip install -e .                 # makes the `anchor` namespace importable

# Build the Rust de-novo scanner (needs a Rust toolchain; static rust-htslib):
cd anchor/caller/rust_pileup
LIBCLANG_PATH=$CONDA_PREFIX/lib cargo build --release    # → target/release/p7_pileup
```

The original study ran on an HPC node with NVIDIA H100 GPUs (pair-HMM scoring uses `numba.cuda`).
A CPU reference pair-HMM (`anchor/allelic/pair_hmm_reference.py`) is provided for environments without
a GPU; pass `cpu` to the self-test to use it.


## 2. Usage

ANCHOR is an interpretable probabilistic stack (no deep learning in the inference path) with **three
engines** that can be run independently. It needs only a position-sorted single-cell long-read BAM
(CB/UB tagged) and two haplotype references; it does **not** require a pre-existing genotype.

| Engine | Directory | What it does | Output |
|--------|-----------|--------------|--------|
| **1. De-novo caller** | `anchor/caller/` | per-UMI molecule×site sign matrix → signed-graph rank-1 two-colouring → 34 features → two-stage LightGBM cascade | phased het-SNP VCF |
| **2. Allelic engine** | `anchor/allelic/` | pair-HMM per-site LLR → Platt ("Route E") calibration → read-level HMM → Beta-binomial UMI aggregation | per-cell / per-gene allele counts (`.h5ad`) |
| **3. Isoform assignment** | `anchor/isoform/` | junction-chain clustering (±10 bp, ≥2 reads) + pair-HMM/HMM + MAP-EM coupling | allele-resolved isoforms (`.h5ad`) |

```
position-sorted scONT BAM (CB/UB tagged) ── wf-single-cell ──┐
                                                             ▼
   [Engine 1] de-novo caller  →  phased het-SNP VCF  ──┐
                                                       ▼
   [Engine 2] allelic engine  →  read → UMI → cell×gene allele counts
                                                       │
   [Engine 3] isoform engine  →  allele × isoform per cell
```

Each engine is a sequence of stage scripts. The canonical stage order and per-stage commands are in
`anchor/docs/pipeline_usage.md`; the full algorithmic specification (factorization objective, pair-HMM
constants, calibration scope) is in `anchor/docs/algorithm_specification.md`. For example, the allelic
engine's calibration step (it resolves its bundled Route E model automatically):

```bash
python anchor/allelic/03_apply_route_e_calibration.py \
    --sample E6.5_LR \
    --input-parquet  raw_llr_E6.5.parquet \
    --output         calibrated_llr_E6.5.parquet \
    --checkpoint     anchor/allelic/route_e/models/adaptive_platt/fold_E6.5_seed42.pt
```


## 3. Example

A small chr19:4–9 Mb example runs the allelic engine (BAM → cell×gene) **and** the isoform engine
end-to-end in ~1–2 min. A CUDA GPU makes stage 2 faster but is **not** required — pass `cpu` to use the
GPU-free scorer (output matches the GPU to float32 rounding). Download the test data archive
(`anchor-testdata-v1.0.0.tar.gz`, distributed separately — see Data/Code availability in the manuscript)
into `tutorial/test_data/data/`, then:

```bash
conda activate anchor
cd tutorial/test_data
ANCHOR_HOME=$(cd ../.. && pwd) PYTHON=$(which python3) bash run_test.sh 0     # GPU 0
# ...or GPU-free:                                                  bash run_test.sh cpu
```

See `tutorial/test_data/README.md` for details.


## 4. Output

```
test_output/
├── cell_gene.h5ad                 # per-cell × per-gene allele counts (≈6,782 cells × 160 genes)
└── isoform/benchmark_*.json       # allele-resolved isoform assignments + benchmark summaries
```

- `cell_gene.h5ad` — AnnData; `layers` hold haplotype-1 (B6 / REF) and haplotype-2 (DBA / ALT) UMI
  counts per cell × gene, with per-site and per-gene allelic-ratio summaries in `.var` / `.uns`.
- `cell_isoform_allelic.h5ad` — per-cell isoform × allele assignment from observed splice structure.

> **Verified state (see `anchor/VERIFICATION.md`):** `import anchor` works; all `.py` files compile;
> the allelic (6 stages) and isoform engines expose working CLIs; the caller models load; Route E
> calibration runs end-to-end on a real checkpoint; the Rust scanner source is byte-identical to the
> binary used in the paper. The de-novo caller's `scripts/` are a research corpus — some run as
> standalone CLIs, the rest reference dataset-specific paths and are provided as the authoritative
> record of how the analysis was run rather than as turnkey CLIs.


## 5. Citation

Fu, Z.C., *et al.* (2026). ANCHOR: de novo, haplotype- and isoform-resolved allelic expression from
single-cell long-read RNA sequencing. Manuscript under review.

Full citation details (journal, volume, DOI) will be added upon publication. See `CITATION.cff`
(inside the archive) for the machine-readable citation.


## 6. License

Copyright © 2026 Zhican Fu and co-authors. All rights reserved.

Free for academic, educational, and not-for-profit research use during peer review. The final
open-source license will be inserted upon publication. See `LICENSE` inside the archive.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT
LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN
NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

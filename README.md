# Benchmarking Protein Function Retrieval: BLAST vs. k-mer vs. ESM2

Final project for **COMPSCI 690U**. We benchmark three families of protein-function retrieval methods — classical sequence alignment, k-mer similarity, and protein language model embeddings — on two [DGEB](https://github.com/TattaBio/DGEB) retrieval tasks, and study where each method wins or fails as a function of sequence similarity to the training corpus.

**Team:** Sameeksha Bhatia, Shreya Arathi, Sivaraaman Balakrishnan, Sergey Semibratov

## Motivation

Retrieving functionally similar proteins is a core primitive in computational biology (annotation transfer, homology search, drug target discovery). The classical tool is BLAST, which relies on local sequence alignment and degrades in the "twilight zone" of low sequence identity. We ask whether dense embeddings from protein language models (ESM2) close that gap, and how much retrieval quality you buy by scaling model size.

## Tasks

| Task | Queries | Corpus | Domain |
|---|---|---|---|
| **Arch Retrieval** | 2,343 archaeal proteins | 9,229 bacterial proteins | cross-domain homology |
| **Euk Retrieval** | 311 eukaryotic proteins | 3,202 bacterial proteins | cross-domain homology |

Both tasks and their relevance judgments (qrels) come from DGEB (`tattabio/arch_retrieval`, `tattabio/euk_retrieval`).

## Methods

1. **BLASTP** (`blast/`) — local alignment search via NCBI BLAST+, top-5 hits per query by e-value/bitscore.
2. **k-mer cosine similarity** (`kmer/`) — sequences vectorized as k-mer count vectors (k = 2, 3, 4) and ranked by cosine similarity. No learned representations, no training data — the "dumb but fast" baseline.
3. **ESM2 embeddings** (`method3_esm2_fixed.py`) — mean-pooled hidden states from three ESM2 checkpoints (35M, 150M, 650M parameters), ranked by cosine similarity. Compared against the 3B-parameter ESM2 result reported in the DGEB paper as a reference upper bound.

All methods are scored with the same evaluation harness (`utils/metrics.py`, and `pytrec_eval` for the DGEB-consistent numbers) so results are directly comparable.

## Results

**MAP@5** (mean average precision at 5), the primary metric:

| Model | Arch Retrieval | Euk Retrieval |
|---|---|---|
| Best k-mer (k=4) | 0.183 | 0.248 |
| BLASTP | 0.299 | 0.340 |
| ESM2-35M | 0.273 | 0.309 |
| ESM2-150M | 0.305 | 0.351 |
| ESM2-650M | **0.314** | **0.359** |
| ESM2-3B *(DGEB paper, reference)* | 0.313 | 0.357 |

Retrieval quality scales cleanly with ESM2 model size, and the 650M-parameter model already matches — and on our runs, slightly exceeds — the published 3B-parameter reference, while comfortably beating both BLASTP and k-mer similarity on this benchmark.

### Twilight-zone breakdown

We also binned queries by their maximum BLAST %identity to the corpus and computed MAP@5 within each bin, to isolate performance on sequences alignment struggles with:

| Similarity bin | BLASTP | k-mer (k=4) | ESM2-650M |
|---|---|---|---|
| Medium (20–40% identity) | 0.273 | 0.085 | **0.294** |
| High (>40% identity) | 0.303 | 0.199 | **0.317** |

ESM2 outperforms BLASTP in both bins, with the largest relative gain in the medium-identity range — evidence that embedding-based retrieval generalizes better than alignment when sequence identity alone is a weak signal. (No queries fell below 20% identity in this corpus, so the true twilight-zone bin is empty here.)

## Repo structure

```
blast/            BLASTP pipeline (run_blast.py, parse_blast.py) + exploration notebooks
kmer/             k-mer vectorization + cosine similarity retrieval, DGEB-consistent eval
method3_esm2_fixed.py   ESM2 embedding extraction, retrieval, scaling + twilight-zone analysis
utils/            Shared metrics (MAP@k) and FASTA I/O helpers
checkpoints/      Saved evaluation outputs (pytrec_eval results, twilight-zone plots)
Scalingplot.py    Generates the ESM2 scaling-law figure (Figure 1)
requirements.txt  Python dependencies
```

## Reproducing

```bash
pip install -r requirements.txt
python main.py              # BLAST pipeline
python kmer/kmer_notebook.py   # k-mer baseline, all k values
# method3_esm2_fixed.py is designed for a GPU runtime (Colab/T4 or better);
# ESM2-650M needs ~6GB VRAM in bfloat16.
```

## Notes / limitations

- BLAST search was run with `-max_target_seqs 5`, so BLASTP precision/recall figures reflect a top-5-only search rather than an exhaustive one.
- The k-mer baseline has no notion of amino acid similarity or evolutionary substitution — it is included as a lower-bound sanity check, not a serious competitor.
- Repository still contains some exploratory/duplicate notebooks (`blast_method1/`, `method3_esm2.ipynb`) from earlier iterations; `method3_esm2_fixed.py` and the results above reflect the corrected, final pipeline.

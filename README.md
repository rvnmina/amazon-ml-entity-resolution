# Amazon ML Challenge 2026 — Business Entity Resolution

Solution for the Business Entity Resolution Challenge: given business records from
3 independent, noisy sources, find which records across sources refer to the same
real-world business. Source 1 is the deduplicated reference; the task is to find all
matching Source 2 / Source 3 records for each Source 1 entity.

## Approach

A blocking + matching pipeline built only on the provided data (no external
databases, APIs, or lookups):

1. **Normalisation** — Unicode/accent to ASCII, leetspeak repair (`N0VA` → nova),
   alias handling (`formerly`, `d/b/a`, `t/a`), domain stripping, legal-suffix
   canonicalisation (Pvt/Private, Ltd/Limited, SARL), street-type and state
   abbreviations, PO-box and store-number removal.
2. **Blocking** — two TF-IDF channels per country and per source (name character
   n-grams + word tokens; address tokens and numbers), with sparse top-K retrieval
   to cut the search space to a small candidate set per Source 1 entity.
3. **Scoring** — each candidate pair scored on blocking cosine similarity, name
   token-set similarity, and shared address numbers.
4. **One-to-one assignment** — each Source 2/3 record is assigned to its single
   best Source 1 entity (the data confirms no record belongs to two entities),
   which removes false merges. A threshold tuned on the precision-heavy F0.5 metric
   decides the final matches.

The pipeline treats `country` as an open set of labels, so it transfers to France
(test-only, unseen in training) without hard-coding.

## Files

| Path | Purpose |
|---|---|
| `src/normalize.py` | Text normalisation for names and addresses |
| `src/pipeline.py` | End-to-end blocking, scoring and output |
| `notebook.ipynb` | Kaggle notebook that runs the full pipeline |
| `requirements.txt` | Pinned dependencies |

## How to run

```bash
pip install -r requirements.txt
python src/pipeline.py --data /path/to/student_resource/dataset --out output
```

Outputs `output/matching_results.tsv` (final matches) and
`output/candidate_pairs.tsv` (the candidate set fed to the matcher).

## Environment

- Python 3.11+
- 16 GB RAM minimum (tested on Kaggle, 4 cores / 30 GB RAM)
- No GPU, no internet at runtime

## Evaluation

Scored with macro-averaged F0.5 (β = 0.5) per Source 1 entity — precision-weighted,
so false merges are penalised more than missed matches.

## Compliance

Uses only the provided training/test data. Libraries: numpy, pandas, scipy,
scikit-learn, rapidfuzz, lightgbm, sparse-dot-topn, anyascii — all permissively
licensed (MIT/BSD/Apache/ISC). No external data, geocoding, or entity-lookup services.

## Note

The dataset is **not** included in this repository (it is challenge-private and
large). Point the pipeline at your local `student_resource/` folder.

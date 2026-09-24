# Gene expression prediction (DNA, cell-type aware)

Predicts how strongly a human gene is expressed in a given cell type or
tissue, from its DNA sequence, as ln(TPM+1).

- [`expression_model_package.ipynb`](expression_model_package.ipynb):
  subscribe, deploy a real-time endpoint, predict, run a batch transform job,
  clean up.
- [`data/input/`](data/input/): sample requests. The same body works for the
  endpoint and for each S3 object in a batch transform job.
- [`data/output/`](data/output/): the response to each sample.
- [`gene_table.tsv`](gene_table.tsv): the TSS, TES and request region of the
  26,821 human genes (GENCODE V29, one transcript per gene) the model was
  built on. Use it to build requests.
- [`cell_types.jsonl`](cell_types.jsonl): the 1,813 cell-type and tissue
  descriptions the model accepts, in the exact text to send as
  `options.description`. Sources are credited in
  [`ATTRIBUTION.md`](ATTRIBUTION.md).
- [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md): third-party components
  in the model package.

## Samples

| Sample | Gene | Cell line |
|---|---|---|
| `APOA1_HepG2` | APOA1, a liver gene | HepG2 (liver) |
| `APOA1_K562` | APOA1 | K562 (erythroleukemia) |
| `HBB_HepG2` | HBB, beta-globin | HepG2 |
| `HBB_K562` | HBB | K562 |

Expected shape: APOA1 high in HepG2 and low in K562, HBB the other way round.

The published responses were produced on `ml.g4dn.xlarge`. The same request
on another GPU instance type can differ by up to about 0.06, because the
accelerators use different reduced-precision arithmetic (fp16 on
`ml.g4dn.xlarge`, bf16 on `ml.g5.xlarge`).

## Request

`Content-Type: application/json`

```json
{
  "sequence": "ACGT...",
  "sequence_name": "APOA1_HepG2",
  "tss_index": 40960,
  "tes_index": 43160,
  "options": {"description": "assay term name is polyA plus RNA-seq. biosample summary is Homo sapiens HepG2."}
}
```

| Field | Required | Meaning |
|---|---|---|
| `sequence` | yes | GRCh38 DNA in the gene's orientation, A, C, G, T, N |
| `tss_index` | yes | 0-based position of the transcription start site in `sequence`; at least 40,960 |
| `tes_index` | yes | Exclusive end of the gene: `sequence[tss_index:tes_index]` is the gene body. Greater than `tss_index`, at most the sequence length |
| `options.description` | yes | The cell type or tissue, exactly as written in the published cell-type list |
| `sequence_name` | no | Free label, echoed back |

To build a request from `gene_table.tsv`: fetch `chrom:region_start-region_end`
from GRCh38, reverse-complement it if `strand` is `-`, then set `tss_index` to
40960 and `tes_index` to the sequence length. Submit the sequence in the
gene's own orientation: only its length is checked, so a wrong-strand
submission returns a confident wrong answer rather than an error.

`tes_index` is **exclusive**: `sequence[tss_index:tes_index]` is the gene
body. An off-by-one there does not error, it just changes the score.

Requests with less than 40,960 bp upstream of the TSS are rejected. **Do not
pad the upstream end with N to satisfy the rule**: an under-filled upstream
arm produces a low score with no error at all. Skip the gene instead. The
`usable` column marks the genes too close to a chromosome end to have 40,960
bp upstream.

## Response

| Field | Meaning |
|---|---|
| `prediction.expression`, `prediction.expression_log_tpm` | Predicted expression, ln(TPM+1) |
| `prediction.expression_tpm` | The same value, as TPM |
| `summary.sequence_length` | Length of the region the model was handed, not of the sequence you submitted |
| `scored_window` | `[start, end)`, half-open, in your submitted sequence's coordinates |

Invalid requests return 400 (wrong alphabet, indices missing or out of range,
unknown field). 415 means the content type was not `application/json`, and
503 means the endpoint is still loading, which takes about 4 minutes.

## Instances

GPU only: `ml.g5.xlarge` (recommended) or `ml.g4dn.xlarge`, for real-time
endpoints and batch transform. Serverless inference is not available for
Marketplace model packages.

## Limitations

Predictions are for cell types in `cell_types.jsonl`; descriptions outside it
are not reliable. For genes longer than about 26 kb the model reads only the
first part of the gene body. It is a human model.

Research use only. Not for diagnostic or clinical use.

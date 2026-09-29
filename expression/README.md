# Gene expression prediction (DNA, cell-type aware)

Predicts how strongly a human gene is expressed in the cell type, tissue or
experimental context you describe, directly from its DNA sequence.

- [`expression_model_package.ipynb`](expression_model_package.ipynb): deploy
  the model from AWS Marketplace, predict on a real-time endpoint, run the same
  requests as a batch transform job, and clean up.
- [`data/input/`](data/input/): sample requests. The same body works for an
  endpoint and for each S3 object in a batch transform job.
- [`data/output/`](data/output/): the response to each sample.

## Samples

Five requests: two genes, each in contexts where it is and is not expected to
be expressed. Each cell context is a detailed description of an RNA-seq
experiment or tissue; descriptions like these, with the assay, the sample and
the conditions, work best. Use them as a pattern for your own contexts.

| Sample | Gene | Cell context | Predicted expression |
|---|---|---|---|
| `APOA1_HepG2` | APOA1, apolipoprotein A-I (made in the liver) | HepG2 liver cell line, polyA RNA-seq | 3,828 TPM |
| `APOA1_liver` | APOA1 | Liver tissue, RNA-seq | 5,232 TPM |
| `APOA1_whole_blood` | APOA1 | Whole blood, RNA-seq | 0.86 TPM |
| `HBB_whole_blood` | HBB, beta-globin (made in red blood cell precursors) | Whole blood, RNA-seq | 7,612 TPM |
| `HBB_liver` | HBB | Liver tissue, RNA-seq | 174 TPM |

The full description text is in each request's `options.description`. The
sequences are GRCh38, in each gene's orientation, from 40,960 bp upstream of
the transcription start site to the end of the gene: APOA1
chr11:116,835,751-116,878,910 (-), HBB chr11:5,226,263-5,268,032 (-). The
responses were produced on `ml.g5.xlarge`; other instance types can differ
slightly.

## Request

`Content-Type: application/json`

```json
{
  "sequence": "ACGT...",
  "sequence_name": "APOA1_HepG2",
  "tss_index": 40960,
  "tes_index": 43160,
  "options": {"description": "assay term name is polyA plus RNA-seq. description is RNA-seq on human HepG2. biosample summary is Homo sapiens HepG2. life stage age is child 15 years. perturbed is False. strand specificity is unstranded. genome annotation is V29. mapped run type is paired-ended. id is ENCFF410NOU. genome is GRCh38."}
}
```

| Field | Required | Meaning |
|---|---|---|
| `sequence` | yes | Human DNA (A, C, G, T, N) containing the gene of interest, in the gene's orientation, with at least 40,960 bp upstream of the transcription start site |
| `tss_index` | yes | 0-based position of the transcription start site in `sequence` |
| `tes_index` | no | 0-based, exclusive end of the gene in `sequence`, if the sequence includes it |
| `options.description` | yes | Free-text description of the cell type, tissue or experimental context, for example "hepatocyte", "erythroblast", and a detailed description of an RNA-seq experiment as in the samples |
| `sequence_name` | no | A label, echoed back |

To build a request, provide a DNA sequence containing the gene of interest, in
the gene's orientation, preferably centred on its transcription start site,
with at least 40,960 bp upstream so the model sees the regulatory context. Set
`tss_index` to the position of the transcription start site. If the sequence
includes the end of the gene, you can also set `tes_index`. The samples here
run from 40,960 bp upstream of the transcription start site to the end of the
gene, so `tss_index` is 40960 and `tes_index` is the length of the sequence.

## Response

| Field | Meaning |
|---|---|
| `prediction.expression_tpm` | Predicted expression, in TPM |
| `prediction.expression_log_tpm` | The same value on a log scale, ln(TPM+1) |
| `scored_window` | `[start, end)` of your sequence that the model read |

400 means the request is invalid, 415 that the content type is not
`application/json`, and 503 that the endpoint is still starting (about 4
minutes).

## Instances

`ml.g5.xlarge` (recommended) or `ml.g4dn.xlarge`, for real-time endpoints and
batch transform.

Research use only. Not for diagnostic or clinical use.

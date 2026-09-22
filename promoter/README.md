# Promoter prediction (DNA, 2000 bp context)

Scores human DNA for promoter activity. The model reads a 2000 bp context
window and returns a promoter probability for each 1000 bp stretch of the
sequence you submit.

- [`promoter_model_package.ipynb`](promoter_model_package.ipynb): subscribe,
  deploy a real-time endpoint, invoke it, run a batch transform job, clean up.
- [`data/input/`](data/input/): sample requests. The same body works for the
  endpoint and for each S3 object in a batch transform job.
- [`data/output/`](data/output/): the response to each sample.
- [`data/sources.json`](data/sources.json): coordinates and a re-fetch URL
  for every sequence.

## Samples

| Sample | GRCh38 interval | Window probabilities | Windows at 0.5 or more |
|---|---|---|---|
| `ACTB_promoter` | chr7:5,529,602-5,531,601 (-) | 0.9833, 0.8295 | 2 |
| `GAPDH_promoter` | chr12:6,533,517-6,535,516 (+) | 0.7955, 0.7904 | 2 |
| `chr3_intergenic` | chr3:80,000,000-80,001,999 (+) | 0.0100, 0.0103 | 0 |

Promoter samples are 1,000 bp upstream of the canonical Ensembl transcription
start site plus 1,000 bp downstream, in the gene's orientation. They show the
request and response shape; they are not an accuracy benchmark.

## Request

`Content-Type: application/json`

```json
{"sequence": "GCGGCCTCCAGATGGTC...", "sequence_name": "ACTB_promoter"}
```

| Field | Required | Constraints |
|---|---|---|
| `sequence` | yes | A, C, G, T, N, any case. 300 to 500,000 bp |
| `sequence_name` | no | Free label, echoed back |

Any other field is rejected with HTTP 400. There is no threshold parameter:
every window's probability is returned, so apply your own cut-off to
`window_details`.

## Response

| Field | Meaning |
|---|---|
| `summary.total_windows` | 1000 bp windows the sequence was cut into |
| `summary.promoter_windows` | Windows with probability 0.5 or more |
| `regions[]` | Those windows, with zero-based half-open `start` and `end` and a `score` |
| `window_details[]` | Every window, with its `probability` and padding |
| `formats.bed`, `formats.bedgraph` | Ready-to-write BED and bedGraph text |

## Invoke a deployed endpoint

```bash
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name <your-endpoint> \
  --content-type application/json \
  --body fileb://data/input/ACTB_promoter.json \
  output.json
```

## Instances

GPU only: `ml.g5.xlarge` (recommended) or `ml.g4dn.xlarge`, for endpoints and
batch transform. Serverless inference is not available for Marketplace model
packages.

## Limitations

The model localises a promoter to a 1000 bp window, not to a transcription
start site, and has no notion of strand. It was trained on human sequence.
Not suitable for sequence containing coding exons.

Research use only. Not for diagnostic or clinical use.

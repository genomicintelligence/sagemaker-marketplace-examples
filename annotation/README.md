# Gene annotation

Finds genes in a DNA sequence and reconstructs the structure of each
transcript: start and end, exons, introns and coding sequence, with each
transcript typed as mRNA or lncRNA. Results come back as JSON, GFF3 or BED.

- [`annotation_model_package.ipynb`](annotation_model_package.ipynb): subscribe,
  deploy a real-time endpoint, annotate as JSON and GFF3, run a batch
  transform job, clean up.
- [`data/input/`](data/input/): sample requests. The same body works for the
  endpoint and for each S3 object in a batch transform job.
- [`data/output/`](data/output/): the JSON and GFF3 response to each sample.
- [`data/sources.json`](data/sources.json): coordinates, a re-fetch URL and
  the Ensembl genes in every region.

## Samples

| Sample | GRCh38 region (+ strand) | Length | Transcripts | mRNA / lncRNA |
|---|---|---|---|---|
| `hbb_10kb` | chr11:5,225,001-5,235,000 | 10,000 bp | 2 | 2 / 0 |
| `brca1_126kb` | chr17:43,044,295-43,170,245 | 125,951 bp | 8 | 2 / 6 |
| `dense_500kb` | chr19:1,000,001-1,500,000 | 500,000 bp | 41 | 41 / 0 |

`hbb_10kb` fits a real-time endpoint. The two larger samples are above the
real-time limit and run through batch transform.

## Request

`Content-Type: application/json`

```json
{"sequence_name": "chr11:5225001-5235000",
 "options": {"reverse_complement": true},
 "sequence": "CTCTATAAGACAACAGAGAC..."}
```

| Field | Required | Constraints |
|---|---|---|
| `sequence` | yes | A, C, G, T, N, any case. 1,000 to 500,000 bp (real-time endpoint: up to 75,000 bp) |
| `sequence_name` | no | Up to 128 characters, no line breaks. Used as the sequence ID in GFF3 and BED |
| `options.reverse_complement` | no | Default `true`: annotate both strands |
| `options.shift_coordinates` | no | `"UCSC"` reports coordinates on the `chrN:start-end` locus named in `sequence_name` |

## Response

Chosen with the `Accept` header:

| Accept | Body |
|---|---|
| `application/json` (default) | `summary`, `transcripts[]` and `formats.gff3` / `formats.bed` |
| `text/x-gff3` | GFF3 only |
| `text/x-bed` | BED only |

Each entry in `transcripts[]` has `start`, `end` (zero-based, half-open),
`strand`, `score`, `tss_position`, `polya_position`, `transcript_type`
(`mRNA` or `lnc_RNA`) with its score, and `exons`, `introns` and `cds`.

Errors: 400 invalid request, 413 a real-time request above 75,000 bp (use
batch transform), 415 unsupported Content-Type or Accept, 503 while the
endpoint is starting.

## Invoke a deployed endpoint

```bash
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name <your-endpoint> \
  --content-type application/json \
  --accept text/x-gff3 \
  --body fileb://data/input/hbb_10kb.json \
  hbb_10kb.gff3
```

## Batch transform

One request per S3 object, `ContentType` `application/json`, `SplitType`
`None`. The default invocation timeout of 600 seconds covers 500,000 bp
records.

## Instances

GPU only: `ml.g5.xlarge` (recommended) or `ml.g4dn.xlarge`, for endpoints and
batch transform. About 5 seconds for a 10 kb gene region on `ml.g5.xlarge`.
Serverless inference is not available for Marketplace model packages.

Research use only. Not for diagnostic or clinical use.

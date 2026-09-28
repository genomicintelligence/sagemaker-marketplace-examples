# Genomic Intelligence on AWS Marketplace: examples

Sample data and notebooks for Genomic Intelligence models sold as Amazon
SageMaker model packages on AWS Marketplace. Each model runs inside your own
AWS account.

| Model | Folder | Listing |
|---|---|---|
| Promoter prediction (DNA, 2000 bp context) | [`promoter/`](promoter/) | Not yet public |
| Gene expression prediction (DNA, cell-type aware) | [`expression/`](expression/) | [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-r5ytjcekc52y6) |

Each folder has sample requests and responses for the real-time endpoint and
batch transform, and a notebook that subscribes, deploys, runs both, and
cleans up.

Research use only. These models are not medical devices and are not for
diagnostic or clinical use.

Support: contact@genomicintelligence.ai

The notebook and code here are MIT-0 (see `LICENSE`). The sample sequences are public GRCh38 data from Ensembl. The model itself is licensed separately, through the AWS Marketplace listing.

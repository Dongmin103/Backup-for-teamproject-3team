# Backup Size Estimate — AWS Exit

Generated from read-only local/AWS inventory checks.

## Local disk
- Available disk: **606 GiB** (`df -h .`)
- Current project size: **21.31 GiB**
- Existing local `RAG_PDF/`: **1.11 GiB**
- New `.aws-exit-backup/`: currently **4.00 KiB**

## Estimated private/off-Git download size

| Group | Estimated size | Notes |
|---|---:|---|
| S3 `say2-3team` selected prefixes | **16.73 GiB** | mostly `raw_data/` + `athena_ready/` |
| ECR compressed images | **9.04 GiB** | Docker save may become ~1.5–2x, budget **18.07 GiB** |
| Lambda code packages | **21.88 MiB** | metadata from Lambda CodeSize |
| SageMaker model artifacts found | **2.01 MiB** | selected artifacts only; one excluded/non-project URI 404 |
| Existing local RAG_PDF | **1.11 GiB** | already on disk, not additional download unless copied |
| DB/RDS/Neptune/OpenSearch | **unknown** | exact size requires export; keep separate budget |

## S3 prefix breakdown

| Prefix | Objects | Size |
|---|---:|---:|
| `web/` | 10 | 2.19 MiB |
| `model/` | 17 | 126.52 MiB |
| `lambda/` | 1 | 60.20 MiB |
| `source/` | 6 | 273.91 KiB |
| `scripts/` | 2 | 7.60 KiB |
| `rag-pdfs/` | 7 | 32.17 MiB |
| `raw_data/` | 374 | 10.74 GiB |
| `neptune-bulk-load/` | 4 | 9.80 KiB |
| `athena_ready/` | 15 | 5.77 GiB |
| `archive/` | 4 | 1.00 MiB |


## Practical disk budget
- **Minimum GitHub-safe package**: likely < 50 MiB.
- **Private S3 selected prefixes only**: about **16.73 GiB**.
- **Private S3 + ECR Docker save budget**: about **34.80 GiB**.
- **With DB/search exports**: unknown; reserve extra **20–100+ GiB** depending on logical dump sizes.

Current 606 GiB free is enough for the known S3/ECR/Lambda/model backup estimate.

## GitHub rule
Do not push raw data, DB dumps, model artifacts, Lambda zips, Docker tarballs, secrets, or PHI-like data. Push only sanitized README/manifests/runbooks/templates.

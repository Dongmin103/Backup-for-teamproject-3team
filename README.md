# Backup for Team Project 3team — StrokePredict AWS Exit

This repository is a **GitHub-safe backup package** for the 3team StrokePredict project before the original AWS account is removed.

It intentionally contains **documentation, sanitized manifests, architecture maps, and restore runbooks only**. It does **not** contain private artifacts such as raw datasets, database dumps, model binaries, Lambda zip packages, Docker image archives, API keys, passwords, or PHI-like data.

## What this repo is for

1. Preserve the structure of the deployed AWS system.
2. Record which AWS resources belonged to this project.
3. Explain how those resources connect to each other.
4. Explain what must be stored in a separate private/off-Git backup bundle.
5. Provide a local-portability path after AWS is gone.

## What this repo is not

This is **not** the full binary/data backup. The following must stay in a private encrypted/off-Git bundle:

- S3 raw data and Athena-ready datasets
- DB dumps and table exports
- OpenSearch/Neptune exports
- Lambda zip packages
- Docker image tarballs
- SageMaker model artifacts
- API keys, passwords, credentials, and secret values
- PHI-like patient data

## High-level architecture

```text
                           Internet / Browser
                                  |
                                  v
                    +-----------------------------+
                    | CloudFront E7XQCFWDVFK4G    |
                    | d1k8g7xdpghccc.cloudfront   |
                    +--------------+--------------+
                                   |
                +------------------+------------------+
                |                                     |
                v                                     v
+--------------------------------+     +-----------------------------------+
| S3 frontend origin             |     | ALB API origin                    |
| bucket: say2-3team             |     | strokepredict-v4-alb             |
| origin path: /web              |     | paths: /api/* /health /healthz   |
+--------------------------------+     +----------------+------------------+
                                                        |
                                                        v
                                      +-------------------------------------+
                                      | ECS backend service                 |
                                      | cluster: stroke-agent-api-http2...  |
                                      | service: strokepredict-v4-backend   |
                                      | container port: 8000                |
                                      +---------+------------+--------------+
                                                |            |
                         +----------------------+            +----------------------+
                         |                                                   |
                         v                                                   v
        +-----------------------------------+              +-----------------------------------+
        | Prediction route                  |              | Guideline/RAG and DDI routes      |
        | backend /api/predict             |              | backend guideline/ddi adapters    |
        +----------------+------------------+              +----------------+------------------+
                         |                                                  |
                         v                                                  |
        +-----------------------------------+                               |
        | API Gateway                       |                               |
        | id: 7o6kyw9542                    |                               |
        | /prod/predict                     |                               |
        +----------------+------------------+                               |
                         |                                                  |
                         v                                                  v
        +-----------------------------------+       +------------------------------------------+
        | Lambda: pre-3team-orchestrator    |       | Lambda: pre-guideline-answer-lambda...  |
        | routes model requests             |       | OpenSearch RAG + LLM answer generation  |
        +-------+-----------+----------+----+       +------------------+-----------------------+
                |           |          |                               |
                v           v          v                               v
+-------------------+ +-------------------+ +-------------------+  +--------------------------+
| SageMaker model 1 | | SageMaker model 2 | | SageMaker model 4 |  | OpenSearch domain        |
| KNHANES/unified   | | MIMIC/ICU         | | IST/ensemble      |  | pre-os-rag-smoke-3team   |
+-------------------+ +-------------------+ +-------------------+  +--------------------------+
                |
                v
+---------------------------------------------------------------+
| Qwen 3.6 restore pack, us-east-1                              |
| endpoint: qwen36awq32k-vllm-endpoint (deleted/not found)      |
| preserve model/config/ECR image for future local/cloud restore |
+---------------------------------------------------------------+

DDI / state side:

+--------------------+      +---------------------------+
| stroke-ddi-check   +----->+ RDS / DynamoDB / Neptune  |
| Lambda/API         |      | DDI drugs/pairs/graph     |
+--------------------+      +---------------------------+

RAG rebuild side:

+--------------------+      +--------------------------+      +---------------------+
| RAG_PDF local dir  +----->+ local embedding/vector   +----->+ local RAG service   |
| PDFs/chunks/md     |      | OpenSearch/FAISS/Chroma  |      | no AWS required     |
+--------------------+      +--------------------------+      +---------------------+
```

## Included primary AWS resources

### Frontend and routing

- CloudFront distribution: `E7XQCFWDVFK4G`
- Domain: `d1k8g7xdpghccc.cloudfront.net`
- S3 frontend bucket/prefix: `s3://say2-3team/web/`
- API origin: `strokepredict-v4-alb-437058920.ap-northeast-2.elb.amazonaws.com`

### ECS backend

- Cluster: `stroke-agent-api-http2-cluster`
- Included service: `strokepredict-v4-backend-service`
- Task definition: `strokepredict-v4-backend:13`
- Container: `strokepredict-v4-backend`
- Port: `8000`

Excluded by user decision:

- `stroke-agent-api-http2-service`

### Prediction path

- Backend route: `/api/predict`
- API Gateway id: `7o6kyw9542`
- Orchestrator Lambda: `pre-3team-orchestrator`
- Included SageMaker endpoints:
  - `analysis-0409-unified-api-ep-20260422-0305`
  - `mimic-hosp-icu-endpoint`
  - `strokepredict-v5-endpoint`
  - `stroke-ensemble-endpoint`

Excluded non-project endpoints:

- `say2-1team-clinsight-prediction`
- `say2-2team-soonet-endpoint`

### Lambda functions to preserve privately

1. `pre-guideline-answer-lambda-3team` — guideline RAG answer
2. `pre-3team-orchestrator` — prediction router
3. `stroke-ddi-check` — DDI check
4. `strokepredict-icu-predictor` — ICU predictor helper
5. `strokepredict-guideline-llm` — legacy guideline LLM
6. `strokepredict-pdf-generator` — PDF generator
7. `say2-3team-ddi-schema-runner` — DDI schema runner

### Stateful resources

- Aurora PostgreSQL: `patient-db-cluster`
- RDS PostgreSQL: `pre-rds-2-3-team-ddi`
- Neptune: `pre-neptune-2-3-team-ddi`
- DynamoDB:
  - `say2-3team-ddi-drugs`
  - `say2-3team-ddi-pairs`
  - `stroke_ddi_drugs`
  - `stroke_ddi_pairs`
  - `strokepredict-icu-patients`
- OpenSearch: `pre-os-rag-smoke-3team`
- Redis: `pre-redis-2-3-team-ddi`

## Disk estimate before private downloads

Known estimate from `manifests/backup-size-estimate.md`:

- Available local disk at estimate time: **606 GiB**
- S3 selected prefixes: **16.73 GiB**
- ECR images compressed: **9.04 GiB**
- ECR `docker save` budget: **~18.07 GiB**
- Lambda code packages: **21.88 MiB**
- Known SageMaker artifacts: **2.01 MiB**
- DB/search exports: unknown until export; reserve **20–100+ GiB** if possible

## Local portability plan

AWS service replacements:

| AWS service | Local/off-AWS replacement |
|---|---|
| CloudFront/S3 frontend | static server or nginx |
| ALB/ECS backend | Docker Compose FastAPI/backend container |
| API Gateway | direct backend route |
| Lambda functions | Python modules or FastAPI routers |
| SageMaker endpoints | local model loaders or explicit degraded stubs |
| Qwen SageMaker | local vLLM if GPU allows; otherwise LLM adapter substitute |
| OpenSearch RAG | local OpenSearch, FAISS, or Chroma rebuilt from `RAG_PDF/` |
| Aurora/RDS | local PostgreSQL |
| DynamoDB | DynamoDB Local, SQLite, or PostgreSQL adapter |
| Neptune | graph export/import substitute, or explicit degraded mode |
| Redis | local Redis |

## Degraded-mode contract

If a local replacement cannot fully restore a feature, the feature must not pretend to work silently. It should return/log:

```json
{
  "degraded": true,
  "missing_dependency": "name of missing artifact/resource",
  "behavior": "stub | partial_restore | local_substitute",
  "restore_instruction": "what artifact/config is needed to restore full behavior"
}
```

## Repository contents

```text
.
├── README.md
├── .env.example
├── docs/
│   ├── architecture-ascii.md
│   ├── private-artifact-policy.md
│   ├── resource-connections.md
│   └── local-portability.md
├── manifests/
│   ├── aws-resource-manifest.sanitized.json
│   ├── backup-size-estimate.json
│   └── backup-size-estimate.md
└── restore/
    └── restore-runbook.md
```

## Restore order

1. Clone this repository.
2. Attach the separate private encrypted backup bundle.
3. Restore frontend assets or serve local static build.
4. Start local backend.
5. Restore/import model artifacts and DB/search exports where available.
6. Rebuild RAG from `RAG_PDF/` if OpenSearch index is not available.
7. Run no-AWS smoke tests.
8. Mark unavailable pieces as explicit degraded mode.

## Safety rule

Never push the private bundle to GitHub.

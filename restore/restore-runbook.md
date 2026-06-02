# Restore Runbook

## 0. Inputs needed

- This GitHub-safe repository.
- Private encrypted backup bundle containing raw exports/artifacts.
- Local machine or server with Docker.
- Optional GPU if restoring Qwen/vLLM locally.

## 1. Validate private bundle

```sh
sha256sum -c SHA256SUMS.txt
```

## 2. Start local services

Prepare local services such as PostgreSQL, Redis, optional OpenSearch/FAISS/Chroma, and the backend container.

## 3. Frontend

Serve the preserved frontend bundle from the private S3 export or rebuild frontend from source if available.

## 4. Backend

Use local environment placeholders from `.env.example`. Do not use AWS credentials in local smoke mode.

## 5. Prediction

Restore SageMaker model artifacts into local loaders where possible. If unavailable, return explicit degraded response.

## 6. RAG

If the AWS OpenSearch index is gone, rebuild from local `RAG_PDF/` sources/chunks. If using a non-Titan embedding model, rebuild the full vector index because vector dimensions may differ.

## 7. DDI/state

Restore RDS/DynamoDB/Neptune exports if available. If not available, mark DDI as degraded.

## 8. No-AWS smoke

Unset AWS credential variables and run:

```sh
env -u AWS_ACCESS_KEY_ID -u AWS_SECRET_ACCESS_KEY -u AWS_SESSION_TOKEN \
  curl http://localhost:8000/health
```

Then verify representative prediction, RAG, and DDI requests.

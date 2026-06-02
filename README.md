# 3팀 Team Project 백업 — StrokePredict AWS 탈출/로컬 이식 문서

이 저장소는 기존 AWS 계정이 삭제되거나 더 이상 접근 불가능해질 상황에 대비해, **3팀 StrokePredict 프로젝트의 AWS 리소스 구조와 복원 방법을 GitHub에 안전하게 남기기 위한 백업 저장소**입니다.

중요: 이 저장소에는 **문서, 정리된 리소스 목록, ASCII 아키텍처, 복원 runbook, 로컬 이식 계획, sanitized manifest만** 포함합니다.

이 저장소에는 아래 항목을 절대 넣지 않습니다.

- DB dump
- 원본/raw 데이터
- PHI/환자정보 가능 데이터
- SageMaker/model artifact
- Lambda zip 파일
- Docker image tar 파일
- API key, password, token, secret value
- 실제 `.env` 파일

실제 원본 백업 파일은 GitHub가 아니라 별도의 **private/off-Git 백업 묶음**에 보관해야 합니다.

---

## 1. 이 저장소의 목적

이 저장소의 목적은 다음과 같습니다.

1. AWS 리소스들이 어떻게 연결되어 있었는지 기록합니다.
2. 어떤 리소스를 백업해야 하는지 남깁니다.
3. GitHub에 올려도 안전한 sanitized manifest를 보관합니다.
4. AWS 없이 로컬에서 다시 실행할 때 어떤 식으로 대체해야 하는지 설명합니다.
5. private 백업 묶음에 무엇이 들어가야 하는지 구분합니다.

---

## 2. 전체 리소스 구조

```text
                           사용자 / 브라우저
                                  |
                                  v
                    +-----------------------------+
                    | CloudFront                  |
                    | ID: E7XQCFWDVFK4G           |
                    | d1k8g7xdpghccc.cloudfront   |
                    +--------------+--------------+
                                   |
                +------------------+------------------+
                |                                     |
                v                                     v
+--------------------------------+     +-----------------------------------+
| S3 프론트엔드 origin           |     | ALB API origin                    |
| bucket: say2-3team             |     | strokepredict-v4-alb              |
| origin path: /web              |     | /api/* /health /healthz           |
+--------------------------------+     +----------------+------------------+
                                                        |
                                                        v
                                      +-------------------------------------+
                                      | ECS 백엔드 서비스                  |
                                      | cluster: stroke-agent-api-http2...  |
                                      | service: strokepredict-v4-backend   |
                                      | container port: 8000                |
                                      +---------+------------+--------------+
                                                |            |
                         +----------------------+            +----------------------+
                         |                                                   |
                         v                                                   v
        +-----------------------------------+              +-----------------------------------+
        | 예측 API 경로                     |              | RAG / DDI 관련 경로              |
        | backend /api/predict             |              | guideline / ddi adapters         |
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
        | 예측 모델 endpoint 라우터         |       | Guideline RAG 답변 생성                 |
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
| endpoint: qwen36awq32k-vllm-endpoint                          |
| 당시 endpoint는 삭제됨/not found 상태였지만                    |
| model/config/ECR image 정보는 재생성용으로 보존해야 함         |
+---------------------------------------------------------------+

DDI / 상태 저장소 쪽:

+--------------------+      +---------------------------+
| stroke-ddi-check   +----->+ RDS / DynamoDB / Neptune  |
| Lambda/API         |      | DDI drugs/pairs/graph     |
+--------------------+      +---------------------------+

RAG 로컬 재구축 쪽:

+--------------------+      +--------------------------+      +---------------------+
| RAG_PDF 로컬 폴더  +----->+ local embedding/vector   +----->+ local RAG service   |
| PDFs/chunks/md     |      | OpenSearch/FAISS/Chroma  |      | AWS 없이 재구축     |
+--------------------+      +--------------------------+      +---------------------+
```

---

## 3. 포함된 주요 AWS 리소스

### 3.1 프론트엔드 / 라우팅

- CloudFront distribution: `E7XQCFWDVFK4G`
- CloudFront domain: `d1k8g7xdpghccc.cloudfront.net`
- S3 frontend bucket/prefix: `s3://say2-3team/web/`
- API origin ALB: `strokepredict-v4-alb-437058920.ap-northeast-2.elb.amazonaws.com`

CloudFront 동작:

```text
기본 경로        -> S3 /web 프론트엔드
/api/*          -> ALB -> ECS backend
/health         -> ALB -> ECS backend
/healthz        -> ALB -> ECS backend
```

### 3.2 ECS 백엔드

- Cluster: `stroke-agent-api-http2-cluster`
- 포함할 서비스: `strokepredict-v4-backend-service`
- Task definition: `strokepredict-v4-backend:13`
- Container: `strokepredict-v4-backend`
- Port: `8000`
- ECR image:
  - `666803869796.dkr.ecr.ap-northeast-2.amazonaws.com/strokepredict-v4-backend:v4-backend-qwen-summary-vpce-20260527T1605KST`

백업 대상에서 제외한 서비스:

- `stroke-agent-api-http2-service`

### 3.3 예측 API 경로

```text
프론트엔드
  -> CloudFront /api/*
  -> ALB
  -> ECS backend /api/predict
  -> API Gateway 7o6kyw9542 /prod/predict
  -> Lambda pre-3team-orchestrator
  -> SageMaker endpoints
```

포함할 SageMaker endpoints:

- `analysis-0409-unified-api-ep-20260422-0305`
- `mimic-hosp-icu-endpoint`
- `strokepredict-v5-endpoint`
- `stroke-ensemble-endpoint`

제외한 비프로젝트 endpoints:

- `say2-1team-clinsight-prediction`
- `say2-2team-soonet-endpoint`

### 3.4 Lambda 함수

private 백업 묶음에는 Lambda zip도 따로 보관해야 합니다. GitHub에는 zip 파일을 올리지 않습니다.

백업 대상 Lambda:

1. `pre-guideline-answer-lambda-3team` — guideline RAG 답변
2. `pre-3team-orchestrator` — 예측 모델 endpoint 라우터
3. `stroke-ddi-check` — DDI/drug interaction 검사
4. `strokepredict-icu-predictor` — ICU predictor helper
5. `strokepredict-guideline-llm` — legacy/simple guideline LLM
6. `strokepredict-pdf-generator` — PDF 리포트 생성
7. `say2-3team-ddi-schema-runner` — DDI schema runner/initializer

### 3.5 S3

주요 bucket:

- `say2-3team`

중요 prefix:

- `web/`
- `model/`
- `lambda/`
- `source/`
- `scripts/`
- `rag-pdfs/`
- `raw_data/`
- `neptune-bulk-load/`
- `athena_ready/` — **제외: 사용자가 백업 불필요하다고 확인**
- `archive/`

#### 제외 확정 prefix

- `s3://say2-3team/athena_ready/`
  - 사유: 사용자가 백업하지 않아도 된다고 확인함.
  - 효과: private S3 백업 예상 용량에서 약 **5.77 GiB** 감소.

### 3.6 DB / 상태 저장소 / 검색

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

### 3.7 Qwen 3.6 restore pack

- Region: `us-east-1`
- Endpoint: `qwen36awq32k-vllm-endpoint`
- 당시 상태: deleted/not found
- Model: `qwen36awq32k-vllm-model`
- Endpoint config: `qwen36awq32k-vllm-endpoint-config`
- Instance type: `ml.g6e.2xlarge`
- ECR image:
  - `666803869796.dkr.ecr.us-east-1.amazonaws.com/sagemaker-qwen-vllm-v3:amd64-vllm-r4-awq32k`
- Digest:
  - `sha256:312c43db4b38c87d11822af7423e731cc9b9f5c99a179d7747730b863327d93d`
- Served model:
  - `cyankiwi/Qwen3.6-27B-AWQ-INT4`

---

## 4. 로컬 디스크 / 백업 용량 산정

산정 당시 로컬 디스크 여유:

- **606 GiB**

GitHub에 올린 문서 패키지:

- 약 **수십 KiB ~ 수백 KiB** 수준

private/off-Git 백업 예상:

| 항목 | 예상 용량 |
|---|---:|
| S3 `say2-3team` 주요 prefix | 제외 전 약 **16.73 GiB**, `athena_ready/` 제외 후 약 **10.95 GiB** |
| ECR compressed image | 약 **9.04 GiB** |
| ECR `docker save` 예산 | 약 **18.07 GiB** |
| Lambda code packages | 약 **21.88 MiB** |
| 확인된 SageMaker model artifacts | 약 **2.01 MiB** |
| DB/RDS/Neptune/OpenSearch | export 전까지 정확 산정 불가 |

실무적으로는 다음 정도를 잡는 것이 안전합니다.

- S3만 백업: `athena_ready/` 제외 후 약 **10.95 GiB**
- S3 + ECR image tar까지 백업: `athena_ready/` 제외 후 약 **29.03 GiB**
- DB/Search export까지 포함: 추가 **20~100+ GiB** 여유 권장

---

## 5. GitHub에 있는 것과 없는 것

### GitHub에 있는 것

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

### GitHub에 없는 것

아래는 반드시 별도 private 백업으로 보관해야 합니다.

```text
private-backup/
├── s3/
│   └── say2-3team/
├── lambda-zips/
├── docker-images/
├── sagemaker-model-artifacts/
├── db-dumps/
├── dynamodb-exports/
├── neptune-exports/
├── opensearch-exports-or-rebuild-info/
└── secrets-encrypted/
```

---

## 6. AWS 없이 로컬에서 대체하는 방향

| 기존 AWS 서비스 | 로컬/비AWS 대체 |
|---|---|
| CloudFront/S3 frontend | static server 또는 nginx |
| ALB/ECS backend | Docker Compose + FastAPI/backend container |
| API Gateway | backend direct route |
| Lambda | Python module 또는 FastAPI router |
| SageMaker endpoint | local model loader 또는 degraded stub |
| Qwen SageMaker | local vLLM GPU 실행 또는 대체 LLM adapter |
| OpenSearch RAG | local OpenSearch / FAISS / Chroma |
| Aurora/RDS | local PostgreSQL |
| DynamoDB | DynamoDB Local / SQLite / PostgreSQL adapter |
| Neptune | graph export/import 대체 또는 degraded mode |
| Redis | local Redis |

---

## 7. RAG 복원

로컬에 `RAG_PDF/`가 있으면 RAG는 재구축 가능성이 높습니다.

중요 파일:

- `RAG_PDF/chunks_v2/child_chunks.jsonl`
- `RAG_PDF/markdown/*.md`
- `RAG_PDF/structured/*.docling.json`
- `RAG_PDF/aws/create_index.py`
- `RAG_PDF/aws/embed_and_index.py`

주의:

- AWS Bedrock Titan embedding을 더 이상 쓰지 않는다면 vector dimension이 달라질 수 있습니다.
- embedding 모델을 바꾸면 기존 index를 그대로 쓰지 말고 새로 rebuild해야 합니다.

---

## 8. Degraded mode 규칙

로컬 복원 시 어떤 기능을 완전히 복원하지 못하면, 조용히 mock처럼 성공시키면 안 됩니다.

반드시 아래처럼 명시해야 합니다.

```json
{
  "degraded": true,
  "missing_dependency": "필요하지만 없는 artifact/resource 이름",
  "behavior": "stub | partial_restore | local_substitute",
  "restore_instruction": "완전 복원을 위해 필요한 파일/설정/작업"
}
```

예시:

```json
{
  "degraded": true,
  "missing_dependency": "stroke-ensemble-endpoint model artifact",
  "behavior": "stub",
  "restore_instruction": "private backup의 sagemaker-model-artifacts/에서 model.tar.gz를 복원한 뒤 local model loader에 연결"
}
```

---

## 9. 복원 순서

1. 이 GitHub 저장소를 clone합니다.
2. 별도 private/off-Git 백업 묶음을 준비합니다.
3. private 백업의 checksum을 검증합니다.
4. frontend bundle 또는 source를 복원합니다.
5. local backend를 실행합니다.
6. model artifact를 local loader에 연결합니다.
7. DB/DynamoDB/Neptune/OpenSearch export를 복원합니다.
8. RAG는 `RAG_PDF/` 기반으로 local vector store를 rebuild합니다.
9. AWS credential 없이 smoke test를 실행합니다.
10. 복원하지 못한 기능은 degraded mode로 명시합니다.

---

## 10. Smoke test 예시

AWS credential 없이 실행해야 합니다.

```sh
env -u AWS_ACCESS_KEY_ID \
    -u AWS_SECRET_ACCESS_KEY \
    -u AWS_SESSION_TOKEN \
    curl http://localhost:8000/health
```

확인할 것:

- frontend가 열린다.
- backend `/health`가 성공한다.
- `/api/predict`가 동작하거나 degraded response를 명확히 반환한다.
- RAG query가 동작하거나 rebuild 필요 상태를 명확히 반환한다.
- DDI lookup이 동작하거나 degraded response를 명확히 반환한다.

---

## 11. 절대 금지

이 저장소에 아래 파일을 추가하지 마세요.

```text
*.zip
*.tar
*.tar.gz
*.parquet
*.csv
*.pkl
*.pt
*.pth
*.onnx
*.db
*.sqlite
*.pem
*.key
.env
```

단, `.env.example`은 placeholder만 포함하므로 허용됩니다.

---

## 12. 현재 상태 요약

- GitHub-safe 문서 백업: 완료
- AWS 리소스 sanitized manifest: 완료
- README ASCII 구조 기록: 완료
- private artifact 실제 다운로드: 별도 단계 필요
- DB/Search export: 별도 단계 필요
- 로컬 완전 실행 검증: 별도 단계 필요


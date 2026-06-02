# Resource Connections

## Browser path

```text
Browser -> CloudFront -> S3 /web frontend
Browser -> CloudFront /api/* -> ALB -> ECS backend
```

## Prediction path

```text
ECS backend /api/predict
  -> API Gateway 7o6kyw9542 /prod/predict
  -> Lambda pre-3team-orchestrator
  -> SageMaker endpoints
```

## Guideline/RAG path

```text
ECS backend guideline route
  -> pre-guideline-answer-lambda-3team
  -> OpenSearch pre-os-rag-smoke-3team
  -> LLM/embedding provider
```

## DDI path

```text
ECS backend DDI route
  -> stroke-ddi-check Lambda
  -> RDS / DynamoDB / Neptune / Redis as configured
```

# Local Portability Notes

The local target is not AWS parity first. It is a no-AWS runnable stack with explicit degraded modes.

Priority:
1. Start frontend and backend locally.
2. Replace API Gateway/Lambda hops with local Python/FastAPI adapters.
3. Replace SageMaker calls with local model loaders or explicit degraded stubs.
4. Rebuild RAG from `RAG_PDF/` using local vector storage.
5. Restore DB/DDI/search data from the private bundle where available.
6. Run smoke tests with AWS credentials unset.

Key backend seams:
- `backend/routers/predict.py`
- `backend/routers/guideline.py`
- `backend/routers/ddi.py`

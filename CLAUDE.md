# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Detailed rules are in `.claude/rules/*.md`. The `.claude/CLAUDE.md` file has the full build, test, lint, and architecture reference — read it first for Go/Vue work.

## Quick Reference

**Backend (Go):** `make build-go` | `make test-short` | `make fmt-go`
**Frontend (Vue 3):** `make build-js` | `make test-js` | `make fmt-js`
**All:** `make build-all` | `make test` | `make fmt`
**Dev environment:** `docker compose up` → http://localhost:2342/ (admin/photoprism)

Single Go test: `go test ./internal/<pkg> -run TestName -count=1`
Single JS test: `make vitest-watch`

## photo_upload_service/ (Python)

A standalone Python microservice layer that adds CLIP-based semantic image search on top of PhotoPrism. Three FastAPI services:

| Service | File | Port | Purpose |
|---|---|---|---|
| CLIP Vision | `clip_vision_api.py` | 8001 | Encodes images to 512-d embeddings via `openai/clip-vit-base-patch32` |
| FAISS Store | `faiss_api.py` | 8003 | ANN index (`IndexFlatIP` + `IndexIDMap`) for cosine similarity search |
| Upload Pipeline | `upload_api.py` | 8004 | Orchestrates: image → CLIP Vision `/embed` → FAISS `/add` |

**Service URLs** are configured via env vars:
- `CLIP_VISION_URL` (default `http://localhost:8001`)
- `FAISS_URL` (default `http://localhost:8003`)
- `FAISS_DIM` (default `512`, match the CLIP model output dim)
- `FAISS_INDEX_PATH` / `FAISS_META_PATH` — where the FAISS index is persisted on disk
- `CLIP_MODEL_ID` (default `openai/clip-vit-base-patch32`)

**Install & run:**
```bash
cd photo_upload_service
pip install -r requirements.txt
python clip_vision_api.py   # port 8001
python faiss_api.py         # port 8003
python upload_api.py        # port 8004
```

**Key API endpoints:**
- `POST /upload` (upload_api) — accepts multipart image, returns `{id, total, filename}`
- `POST /embed` (clip_vision_api) — accepts multipart image, returns `{embedding, dim}`
- `POST /add` / `POST /search` / `DELETE /remove/{id}` (faiss_api) — vector store CRUD
- `GET /health` on each service — checks upstream connectivity

**Architecture notes:**
- FAISS index uses inner-product (cosine) similarity; vectors are L2-normalized before storage and at query time.
- The FAISS service is single-process with a `threading.Lock` — not safe for multi-worker deployments without an external lock.
- Index state persists to `faiss.index` + `faiss_meta.json` on every write; call `POST /save` explicitly if you want a manual checkpoint.
- The Upload service is stateless and can be scaled horizontally; the FAISS and CLIP services are stateful/heavy.

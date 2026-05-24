# Mayda Infra

**Global orchestration for the Mayda restaurant management platform.**

One `docker compose up` to run everything: PostgreSQL, Backend API, AI service, and automatic database seeding with authentic Algerian cuisine data.

## Architecture

```
┌──────────┐     ┌───────────────┐     ┌──────────┐
│ postgres │────▶│ mayda-backend │◀────│ mayda-ai │
│  :5432   │     │    :8001      │     │  :8101   │
└────┬─────┘     └───────┬───────┘     └────┬─────┘
     │                   │                  │
     └───────────────────┼──────────────────┘
                         │
                    ┌────▼────┐
                    │  seed   │  (one-shot)
                    └─────────┘
```

| Service | Image | Port | Description |
|---|---|---|---|
| `postgres` | `postgres:15-alpine` | 5432 | PostgreSQL database |
| `mayda-backend` | `ghcr.io/mayda-enigma/mayda-backend:latest` | 8001 | FastAPI restaurant management API |
| `mayda-ai` | `ghcr.io/mayda-enigma/mayda-ai:latest` | 8101 | AI recommendations, search, voice, forecasting |
| `seed` | build locally | — | One-shot data initializer (Algerian food in French) |

## Quick Start

```bash
# 1. Clone and configure
cp .env.example .env
# Edit .env — at minimum set:
#   POSTGRES_PASSWORD, SECRET_KEY, SERVICE_TOKEN

# 2. Start everything
docker compose up -d

# 3. Check health
docker compose ps
curl http://localhost:8001/health    # backend
curl http://localhost:8101/api/health  # ai

# 4. Seed data runs automatically (idempotent — only on first run)
docker compose logs seed
```

## Seed Data

The seed container (`mayda-seed`) runs once after all services are healthy and populates both databases with **authentic Algerian cuisine data in French**:

- **4 restaurants**: Le Jardin d'Alger, El Bahia, L'Oasis du Sud, Dar El Kenza
- **68 Algerian dishes**: Couscous royal, Rechta, Chakhchoukha, Bourek, Baklawa, etc.
- **50+ inventory items**: Semoule, Huile d'olive, Pois chiches, Dattes Deglet Nour, etc.
- **Users**: Admin, managers, waiters, chefs, clients — all with Algerian names
- **50+ seeded orders** with timestamps spanning the last 28 days (for analytics)
- **Reviews, reservations, promotions, loyalty cards** — full demo dataset
- **AI food items**: 50 Algerian ingredients in the forecasting database

All data is idempotent — the seed script checks for existing data and skips if already present.

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `POSTGRES_PASSWORD` | ✅ | — | PostgreSQL password |
| `SECRET_KEY` | ✅ | — | JWT signing key (generate with `openssl rand -hex 32`) |
| `SERVICE_TOKEN` | ✅ | — | Shared secret for inter-service auth |
| `ENVIRONMENT` | | `production` | Runtime environment |
| `LOG_LEVEL` | | `INFO` | Logging verbosity |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | | `30` | JWT token lifetime |
| `REFRESH_TOKEN_EXPIRE_DAYS` | | `7` | Refresh token lifetime |
| `BACKEND_CORS_ORIGIN_REGEX` | | `.*` | CORS allowed origins regex |
| `GEMINI_API_KEY` | | — | Google Gemini API key |
| `OPENAI_API_KEY` | | — | OpenAI API key (fallback) |
| `LLM_PROVIDER` | | `gemini` | LLM provider selection |
| `WHISPER_MODEL` | | `tiny` | Whisper model size |
| `TWILIO_*` | | — | Twilio SMS (optional) |
| `GUIDINI_*` | | — | Guidini integration (optional) |

## Dokploy Deployment

This repository is designed for direct deployment via **Dokploy**:

1. Create a new **Docker Compose** application in Dokploy
2. Connect your Git repository (this one)
3. Set the **environment variables** in the Dokploy UI
4. Deploy — Dokploy will:
   - Clone the repository
   - Build the seed container
   - Pull `ghcr.io/mayda-enigma/mayda-backend:latest` and `mayda-ai:latest`
   - Start `postgres` → `mayda-backend` → `mayda-ai` → `seed` (in order)
   - Data persists across deployments via named volumes

**Important for Dokploy:**
- All volumes use **named volumes** (backup-compatible via Dokploy's Volume Backups)
- Do NOT use absolute host paths in volumes (Dokploy cleans them between deployments)
- Use the **Docker Compose** mode (not Stack)
- Environment variables set in the Dokploy UI are written to `.env` automatically

## Manual Database Seeding

If you need to re-seed:

```bash
# Stop and remove volumes to reset
docker compose down -v

# Or run the seed container manually
docker compose run --rm seed
```

## Volumes

| Volume | Mount | Purpose |
|---|---|---|
| `mayda_postgres_data` | `/var/lib/postgresql/data` | PostgreSQL data |
| `mayda_ai_inventory_data` | `/data` | SQLite DB + joblib models |
| `mayda_ai_chroma_data` | `/chroma` | ChromaDB vector store |
| `mayda_ai_hf_cache` | `/cache/hf` | HuggingFace model cache |
| `mayda_api_logs` | `/app/logs` | Backend API logs |

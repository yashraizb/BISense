# BISense

AI-powered spreadsheet BI insights. Upload CSV or Excel files and ask natural language questions to get instant business intelligence.

## Architecture

```
User (Browser)
  ↕ WebSocket
FastAPI (LangChain + DuckDB embedded)
```

## Prerequisites

- Node.js 20+
- Python 3.11+
- Docker + Docker Compose
- An Anthropic API key

## Quick Start

```bash
# 1. Clone and enter the repo
git clone <repo-url>
cd BISense

# 2. Set up environment
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY

# 3. Run with Docker (2 services)
docker compose up --build

# 4. Open http://localhost:5173
```

## Local Development (without Docker)

### Backend
```bash
cd backend
python -m venv .venv
.venv\Scripts\activate   # Windows
pip install -r requirements.txt
cp .env.example .env     # fill in values
uvicorn app.main:app --reload
```

### Frontend
```bash
cd frontend
npm install
cp .env.example .env.local  # fill in values
npm run dev
```

## Project Structure

```
BISense/
├── frontend/       React + TypeScript + Vite
├── backend/        FastAPI + LangChain + DuckDB (embedded)
└── docker-compose.yml
```

## Deployment

| Component | Platform | Notes                         |
|-----------|----------|-------------------------------|
| Frontend  | Vercel   | Root dir: `frontend/`         |
| Backend   | Railway  | Persistent volume for uploads |

# Autonomous Visual Researcher setup

Get the app running and watch it investigate an image. Allow about five minutes. No API key is needed: it ships with three prepared subjects and a simulated document collection.

## What you need

- Python 3.12 or newer
- Node.js 20 or newer

## 1. Start the API

```bash
cd backend
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

Check it: open <http://localhost:8000/health>. You should see `"status": "ok"`.

## 2. Start the UI

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open <http://localhost:5173>. The UI proxies `/api` to port 8000, so there is nothing to configure.

## 3. Run your first investigation

1. Pick one of the prepared subjects (skyscraper, woodblock print or sports car).
2. Watch the plan, searches and evidence build up.
3. Open the report. Every statement carries a numbered citation; select one to see the source and the claim behind it.
4. Ask a follow-up in "Ask the graph". It is answered from research already done.

## Turn on live research (optional)

Demo mode stays on until all three keys are set. Copy `.env.example` to `.env`, fill it in, and load it into the terminal that starts the API. The backend reads real environment variables; it does not load `.env` by itself.

```bash
cp .env.example .env               # from the project root
# edit .env, then:
set -a; source .env; set +a
cd backend && uvicorn app.main:app --port 8000
```

```env
USE_MOCK_MODE=false
VISION_API_KEY=your-anthropic-key
LLM_API_KEY=your-anthropic-key
SEARCH_API_KEY=your-tavily-key
```

`GET /health` reports `"mock_mode": false` once live mode is active.

## Settings worth knowing

| Variable | Default | What it does |
|---|---|---|
| `MAX_RESEARCH_ITERATIONS` | `5` | How many plan-search-verify rounds one investigation may take. |
| `MAX_SEARCHES` | `20` | Hard cap on web searches per investigation. The main cost control. |
| `MAX_SOURCES` | `50` | Hard cap on sources read per investigation. |
| `DATABASE_PATH` | `./data/research.sqlite3` | Where finished investigations are archived. |
| `CORS_ORIGINS` | `http://localhost:5173,http://127.0.0.1:5173` | Origins allowed to call the API. Set this when the UI is served from another host. |

The API runs at most 4 investigations at once and answers 429 when all slots are busy.

## Run everything with Docker (optional)

```bash
docker compose up --build
```

API on 8000, UI on 5173.

## Run the tests

```bash
cd backend
python -m pytest -q
```

## If something goes wrong

| Symptom | Fix |
|---|---|
| The UI loads but shows no prepared subjects | The API is not running on port 8000. |
| Still in demo mode after adding keys | All three keys must be set, `USE_MOCK_MODE` must be `false`, and the variables must be exported in the API's terminal. |
| 429 "investigations are already running" | Wait for one to finish. The limit is 4 at a time. |
| 413 on upload | The image is over 12 MB. |
| An uploaded image returns almost nothing in demo mode | Demo mode only has documents for the three prepared subjects. Use live mode for your own images. |

More detail: [README.md](README.md), [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md), [docs/ENGINEERING.md](docs/ENGINEERING.md).

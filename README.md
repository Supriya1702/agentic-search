# Agentic Search

A full-stack system that accepts a natural-language topic query, searches the web, scrapes relevant pages, and uses an LLM to extract structured, source-traced entity data — returned as a table.
Demo Link : https://drive.google.com/file/d/16Hno-WOQyH6PqBdPkOVzm2K7KtNvEA3v/view?usp=drivesdk
---

## Approach

The pipeline has four stages:

```
User Query → Web Search (Firecrawl) → Page Scraping (Firecrawl) → Entity Extraction (LLM) → Structured Table
```

1. **Search** — The user's query is sent to Firecrawl's search API, which returns the top 3 relevant URLs.
2. **Scrape** — Each URL is scraped via Firecrawl into clean Markdown, stripping navigation, ads, and boilerplate.
3. **Extract** — The Markdown content and source URL are passed to a local LLM (Llama 3.2 via Ollama). The LLM is prompted to extract entities into a table format, including name, key features, and a `source_quote` for traceability.
4. **Serve** — Results are stored in-memory and served to the React frontend, which renders them in a structured view.

The backend uses FastAPI's `BackgroundTasks` so the HTTP response returns immediately with a `task_id`. The frontend polls `/results/{task_id}` until the pipeline completes.

---

## Design Decisions

### Why Firecrawl for both search and scraping?
Firecrawl provides a single SDK that handles both web search and page scraping with Markdown output. This avoids the complexity of stitching together separate search APIs (Brave, SerpAPI) with separate scraping tools (BeautifulSoup, Playwright). The Markdown output is LLM-friendly — cleaner and more token-efficient than raw HTML.

**Trade-off**: Firecrawl's free tier has credit limits. A production system might use a cheaper search API and only use Firecrawl for scraping.

### Why a local LLM (Ollama + Llama 3.2)?
Running the LLM locally eliminates per-request API costs and avoids rate limits, making the system free to run after initial setup. The OpenAI-compatible API interface means swapping to a cloud model (GPT-4o-mini, Claude) requires changing only the base URL, API key, and model name — no code changes.

**Trade-off**: Llama 3.2 (3B) is less precise at structured extraction than larger models. It occasionally produces malformed JSON or misses entities. A production system would benefit from GPT-4o-mini or a larger local model.

### Why background tasks with polling?
The full pipeline (search + 3 scrapes + 3 LLM calls) takes 30–90 seconds. Blocking the HTTP request would cause timeouts. The background task pattern lets the frontend show immediate feedback and poll for progress.

**Trade-off**: Polling adds ~1s of latency to UI updates. WebSockets would be more responsive but add deployment complexity.

### Why in-memory storage?
For a prototype, a Python dictionary is the simplest possible storage. It avoids external dependencies (Redis, Postgres) and makes the system runnable with a single command.

**Trade-off**: Results are lost on server restart, and the system can't scale horizontally. A production version should use Redis or a database.

### Entity extraction prompt design
The LLM is prompted to extract entities into a table format with `name`, `key features`, and a `source_quote`. The source quote is a verbatim excerpt from the scraped content, making each extracted value traceable to its origin — a key requirement of the challenge.

### Error handling for unsupported sites
Some websites block automated scraping or aren't supported by Firecrawl. The pipeline catches `WebsiteNotSupportedError` and general exceptions per-URL, logging the failure and continuing to the next page rather than aborting the entire search.

---

## Known Limitations

1. **No deduplication** — If the same entity appears across multiple scraped pages, it will appear multiple times in the results. A production system would deduplicate by entity name (with fuzzy matching).

2. **Fixed search limit** — The system scrapes the top 3 results. Some queries would benefit from more sources; others waste time on irrelevant pages. Adaptive result counts or relevance filtering would improve this.

3. **No caching** — Repeated identical queries re-run the entire pipeline. A URL-level cache would significantly reduce latency and cost for overlapping queries.

4. **Extraction quality varies** — Llama 3.2 (3B) sometimes returns poorly structured output, especially on long or complex pages. Larger models produce better results but require more resources or API costs.

5. **No concurrent scraping** — Pages are scraped and processed sequentially. Concurrent scraping would reduce total latency roughly proportionally to the number of URLs.

6. **In-memory storage** — Results don't persist across restarts. Not suitable for production.

7. **No rate limiting** — Multiple simultaneous users could overwhelm the Ollama instance or Firecrawl quota.

---

## Setup Instructions

### Prerequisites

- **Python 3.10+**
- **Node.js 18+**
- **Ollama** — [install from ollama.com](https://ollama.com)
- **Firecrawl API key** — [free tier at firecrawl.dev](https://www.firecrawl.dev/)

### 1. Clone the repository

```bash
git clone https://github.com/Supriya1702/agentic-search.git
cd agentic-search
```

### 2. Install dependencies

```bash
# Python
pip install -r requirements.txt

# Frontend
npm install
```

### 3. Pull the LLM model

```bash
ollama pull llama3.2
# Verify it's running:
ollama list
```

### 4. Configure API keys

Open `main.py` and replace the Firecrawl API key:

```python
firecrawl = Firecrawl(api_key="YOUR_FIRECRAWL_KEY")
```

Or set it as an environment variable and update the code accordingly.

### 5. Build the frontend

```bash
npm run build
```

This compiles the React app into the `static/` directory that FastAPI serves.

### 6. Run the server

```bash
uvicorn main:app --reload
```

Visit **http://localhost:8000**

### Development mode (with hot reload)

```bash
# Terminal 1 — backend
uvicorn main:app --reload --port 8000

# Terminal 2 — frontend dev server (proxies API calls to :8000)
npm run dev
```

Visit **http://localhost:5173**

### Using a cloud LLM instead of Ollama

To use OpenAI instead of a local model, change these lines in `main.py`:

```python
client = OpenAI(base_url="https://api.openai.com/v1", api_key="sk-your-key")
# and in the completion call:
model="gpt-4o-mini"
```

---

## Tech Stack

| Layer     | Technology            | Why                                              |
|-----------|-----------------------|--------------------------------------------------|
| Backend   | FastAPI + Python      | Async-ready, auto-generated docs, fast to build  |
| Search    | Firecrawl Search API  | Combined search + scrape in one SDK              |
| Scraping  | Firecrawl Scrape      | Clean Markdown output, handles JS-rendered pages |
| LLM       | Ollama (Llama 3.2)    | Free, local, OpenAI-compatible API               |
| Frontend  | React + Vite          | Fast dev experience, simple build pipeline        |

---

## Project Structure

```
├── main.py              # FastAPI backend — routes, background pipeline, LLM extraction
├── requirements.txt     # Python dependencies
├── package.json         # Node dependencies and build scripts
├── index.html           # Vite entry HTML
├── src/
│   └── main.jsx         # React frontend
├── static/              # Built frontend (generated by npm run build)
└── README.md
```

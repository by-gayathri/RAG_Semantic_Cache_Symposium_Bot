# Symposium RAG Bot

A simple Retrieval-Augmented Generation (RAG) chatbot built with Python, FastAPI, and Streamlit.

## Project Structure

```
Symposium_Bot/
├── data/
│   └── symposium.pdf              # Source document
├── vector_store/                   # FAISS index (created after ingestion)
├── phase1_ingestion/
│   └── ingest.py                  # Data ingestion script
├── phase2_retrieval/
│   ├── backend.py                 # FastAPI backend (with semantic cache)
│   └── frontend.py                # Streamlit UI (with cache hit badge)
├── .env                           # API keys + Valkey config
├── requirements.txt               # Dependencies
└── README.md                      # This file
```

## Setup Instructions

### 1. Install Dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure Environment Variables

Edit the `.env` file and add your OpenAI API key and Valkey connection details:

```
OPENAI_API_KEY=sk-your-actual-api-key-here

# Valkey connection (semantic cache backend)
VALKEY_HOST=localhost
VALKEY_PORT=6379
```

### 3. Start Valkey (Semantic Cache Backend)

Run Valkey with the bundled search module using Docker (required for semantic caching):

```bash
docker run -d \
  --name valkey-semantic-cache \
  -p 6379:6379 \
  valkey/valkey-bundle:latest
```

Verify it's ready:
```bash
docker exec -it valkey-semantic-cache valkey-cli FT._LIST
# Should return: (empty array)
```

To stop/start later:
```bash
docker stop valkey-semantic-cache
docker start valkey-semantic-cache
```

### 4. Run Phase 1: Data Ingestion

This step needs to be run only once to create the vector store:

```bash
python phase1_ingestion/ingest.py
```

**What it does:**
- Loads the PDF from `data/symposium.pdf`
- Chunks it into 1000-character pieces
- Creates embeddings using OpenAI's text-embedding-3-small
- Stores vectors in `vector_store/` folder

### 5. Run Phase 2: Backend + Frontend

**Terminal 1 - Start Backend API** (from project root):

```bash
python3 -m uvicorn phase2_retrieval.backend:app --reload
```

Backend will run on: http://localhost:8000

**Terminal 2 - Start Streamlit Frontend:**

```bash
python3 -m streamlit run phase2_retrieval/frontend.py
```

Frontend will open in your browser automatically (usually http://localhost:8501)

## How It Works

1. **User asks a question** in the Streamlit chat interface
2. **Frontend sends question** to FastAPI backend
3. **Backend checks the semantic cache** (Valkey via BetterDB)
   - **Cache hit:** Returns the stored answer instantly — no LLM call, no cost
   - **Cache miss:** Continues to retrieval + generation
4. **Backend performs semantic search** to find top 3 relevant PDF chunks
5. **Backend calls GPT-3.5-turbo** with the context and question
6. **Response is stored in the cache** for future similar questions
7. **Response is displayed** in the chat interface — with a `⚡ Cached` badge if served from cache

### Semantic Cache — Why It Matters

Standard caching only catches exact matches. The semantic cache catches **paraphrases**:

| Question | Result |
|---|---|
| "What is the event date?" | CACHE MISS → calls LLM |
| "What is the event date?" (repeated) | CACHE HIT — exact match |
| "When does the event take place?" | CACHE HIT — paraphrase detected |

The backend prints real-time cache stats after every request:
```
[CACHE MISS] "What is the event date?"
[STATS]      total=1 | hits=0 | misses=1 | hit_rate=0.0% | saved=$0.0000

[CACHE HIT]  similarity=0.9700  "When does the event take place?"
[STATS]      total=2 | hits=1 | misses=1 | hit_rate=50.0% | saved=$0.0008
```

A `/stats` endpoint is also available at `http://localhost:8000/stats` for a full summary.

## Technology Stack

- **PDF Processing**: PyPDFLoader
- **Chunking**: RecursiveCharacterTextSplitter (1000 chars, 200 overlap)
- **Embeddings**: OpenAI text-embedding-3-small
- **Vector Store**: FAISS (local)
- **LLM**: GPT-3.5-turbo
- **Semantic Cache**: BetterDB (`betterdb-semantic-cache`) backed by Valkey
- **Cache Store**: Valkey with valkey-search module (`valkey/valkey-bundle` Docker image)
- **Backend**: FastAPI (async)
- **Frontend**: Streamlit

## Notes

- Semantic cache threshold is set to `0.15` (cosine distance) — questions within 85% similarity return a cache hit
- Cache stats (hit rate, cost saved) are printed in the backend terminal after every request and available at `GET /stats`
- The Valkey Docker container must be running before starting the backend

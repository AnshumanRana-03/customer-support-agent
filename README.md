# Customer Support Agent Copilot

An AI-assisted workspace for support teams to manage customer tickets and prepare replies with relevant context. Instead of generating a response from the ticket alone, the copilot can retrieve information from a support knowledge base, look up customer-related details through tools, and use saved resolution history to help draft a more informed reply.

The goal is to help a support agent get to a useful first draft faster—not to remove the agent from the decision.

## What it does

- **Ticket management:** Create tickets and view the existing ticket queue, including customer details, priority, and status.
- **AI reply drafts:** Generate a suggested response for a ticket and save it for review.
- **Knowledge-base retrieval (RAG):** Search support documents for relevant passages and use them as context for draft generation.
- **Customer memory:** Retrieve relevant past context and save accepted resolutions so later requests can benefit from previous support history.
- **Support tools:** Look up a customer's plan/SLA details and check their open-ticket load.
- **Draft review:** Edit a suggested reply, then accept it or leave it pending. Accepting a draft marks the related ticket as resolved and attempts to save the resolution to memory.
- **Context visibility:** Inspect retrieved knowledge, memory matches, tool calls, and related context recorded for a draft.
- **API-first design:** The backend exposes REST endpoints through FastAPI, while the Streamlit dashboard provides the support-agent interface.

## Tech stack

| Component | Role |
|---|---|
| Python | Application logic |
| FastAPI + Uvicorn | REST API and application server |
| Streamlit | Support-agent dashboard |
| LangChain agent | Orchestrates the model and support tools |
| Groq API | Runs the chat model used for draft generation |
| ChromaDB | Persistent vector storage for knowledge retrieval and Mem0 memory |
| Mem0 | Customer-context and resolution memory |
| SQLite | Stores customers, tickets, and draft records |
| Pydantic / pydantic-settings | Request validation and environment-based configuration |
| Google Gemini embeddings (optional/configurable) | Embeddings for the knowledge base and memory when configured |

## How a draft is generated

1. A support agent creates a ticket or selects an existing one.
2. The copilot uses the ticket subject and description to search the support knowledge base and relevant customer memory.
3. A LangChain agent can call support tools to retrieve plan/SLA information and open-ticket load.
4. The model uses the available context to prepare a suggested response.
5. The draft and context summary are stored. The agent can review or edit the draft before accepting it.
6. When a draft is accepted, the ticket is marked resolved and the accepted resolution is offered to the memory layer for future retrieval.

```text
Support Agent
     |
     v
Streamlit Dashboard
     |
     v
FastAPI REST API
     |
     +------ SQLite
     |       customers / tickets / drafts
     |
     v
Support Copilot (LangChain + Groq)
     |
     +------ ChromaDB knowledge search (RAG)
     |
     +------ Mem0 customer memory
     |
     +------ Support tools
             plan/SLA lookup + open-ticket load
```

## Project structure

```text
.
├── app.py                         # Streamlit dashboard
├── main.py                        # FastAPI entry point
├── customer_support_agent/
│   ├── api/
│   │   └── routers/               # Ticket, draft, knowledge, memory, health APIs
│   ├── core/                      # Settings and path configuration
│   ├── integrations/
│   │   ├── memory/                # Mem0 integration
│   │   ├── rag/                   # ChromaDB knowledge-base retrieval
│   │   └── tools/                 # Tools available to the agent
│   ├── repositories/
│   │   └── sqlite/                # SQLite persistence
│   ├── schemas/                   # API request/response models
│   └── services/                  # Copilot, draft, and knowledge services
├── knowledge_base/                # Support articles used for retrieval
├── tests/                         # Tests
├── pyproject.toml                 # Dependencies and project configuration
└── uv.lock                        # Locked dependency versions
```

## Run locally

### 1. Requirements

- Python **3.11** (the project currently specifies `>=3.11,<3.12`)
- [uv](https://docs.astral.sh/uv/) for dependency management
- API keys for the services you configure below

### 2. Clone the repository

```bash
git clone https://github.com/AnshumanRana-03/customer-support-agent.git
cd customer-support-agent
```

### 3. Install dependencies

```bash
uv sync
```

If you prefer a regular virtual environment and `pip`, install the dependencies listed in `pyproject.toml` in a Python 3.11 environment. `uv sync` is the repository's intended setup because it follows `uv.lock`.

### 4. Configure environment variables

Create a `.env` file in the project root. Add your own keys; **do not commit this file or publish API keys**.

```dotenv
GROQ_API_KEY=your_groq_api_key

# Optional: used for Gemini embeddings
GOOGLE_API_KEY=your_google_api_key

# Optional alternative embedding provider for Mem0
# OPENAI_API_KEY=your_openai_api_key

# Optional: use local Hugging Face embeddings for Mem0 instead
# ENABLE_LOCAL_EMBEDDINGS=true

# Optional model/configuration overrides
# GROQ_MODEL=llama-3.1-8b-instant
# LLM_TEMPERATURE=0.2
# GOOGLE_EMBEDDING_MODEL=gemini-embedding-001

# Dashboard setting; use the backend URL if it is not running locally
# API_BASE_URL=http://localhost:8000
```

`GROQ_API_KEY` is required to generate AI drafts. Mem0 also needs an embedding provider configured: Google, OpenAI, or local embeddings enabled with `ENABLE_LOCAL_EMBEDDINGS=true`. The knowledge-base integration uses Gemini embeddings when `GOOGLE_API_KEY` is set; otherwise it uses ChromaDB's default embedding function.

### 5. Start the API

In one terminal:

```bash
uv run uvicorn main:app --host 127.0.0.1 --port 8000 --reload
```

- API health check: `http://127.0.0.1:8000/health`
- Interactive API docs: `http://127.0.0.1:8000/docs`

### 6. Start the dashboard

In a second terminal:

```bash
uv run streamlit run app.py
```

Open the local URL printed by Streamlit. The dashboard defaults to `http://localhost:8000` for its API. If your API is hosted elsewhere, set `API_BASE_URL` in the environment before starting Streamlit.

### 7. Index the support knowledge base

The repository includes example banking-support articles in `knowledge_base/`. After the API is running, ingest them so they can be retrieved during draft generation:

```bash
curl -X POST "http://127.0.0.1:8000/api/knowledge/ingest" \
  -H "Content-Type: application/json" \
  -d '{"clear_existing": false}'
```

The response reports the number of files and chunks indexed. To rebuild the collection from the current documents, set `clear_existing` to `true`.

## API quick reference

The interactive Swagger page at `/docs` is the best place to inspect request schemas and try endpoints. The main routes include:

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | Health check |
| `POST` | `/api/tickets` | Create a ticket; can request automatic draft generation |
| `GET` | `/api/tickets` | List tickets |
| `GET` | `/api/tickets/{ticket_id}` | Get a ticket |
| `POST` | `/api/tickets/{ticket_id}/generate-draft` | Generate and store a draft |
| `GET` | `/api/drafts/{ticket_id}` | Get the latest draft for a ticket |
| `PATCH` | `/api/drafts/{draft_id}` | Edit a draft or update its status |
| `POST` | `/api/knowledge/ingest` | Index the knowledge-base documents |

Additional routes are available for memory search and related support workflows; inspect `/docs` for the complete, current API schema.

## Data and configuration

- **SQLite database:** Defaults to `data/support.db`.
- **Knowledge-base documents:** Markdown and text files in `knowledge_base/`.
- **ChromaDB persistence:** Separate local directories are configured for knowledge retrieval and Mem0 memory under `data/`.
- **Environment settings:** Centralized in `customer_support_agent/core/settings.py`; paths and model settings can be overridden with environment variables.

The local database and vector-store directories are runtime data. Back them up separately if you need to preserve local tickets or memory between environments.

## Testing

Run the test suite with:

```bash
uv run pytest
```

## Current scope and limitations

- This is an **agent-assist** application: generated replies should be reviewed by a human before they are sent to a customer.
- Results depend on the configured model, available knowledge-base content, embedding provider, and stored history.
- The included plan/SLA lookup is a demonstration tool with deterministic sample plan assignment, not a connection to a real billing or subscription system.
- The included knowledge-base articles are sample support content and should be replaced or extended with documents relevant to the intended organization.
- Do not put real customer data or secrets into a public demo environment. Add authentication, authorization, and appropriate data-handling controls before using it with real customer records.

## Possible next improvements

- Add authentication and role-based access for support agents.
- Connect plan and SLA tools to a real customer or billing system.
- Add stronger evaluation for draft quality, retrieval relevance, and tool-call correctness.
- Improve observability and audit logging for production support workflows.

---

Built as a project to explore how retrieval, memory, and tool-using agents can support a practical customer-service workflow.

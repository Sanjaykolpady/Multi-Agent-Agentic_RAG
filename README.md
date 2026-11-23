# Brand Guardian AI

A multi-agent, agentic RAG pipeline that audits YouTube videos for brand-compliance risks. The system downloads a video, indexes it with Azure Video Indexer, retrieves relevant compliance rules from Azure AI Search, and uses Azure OpenAI to produce a structured audit report.

## Architecture

```text
YouTube URL
    |
    v
LangGraph Workflow
    |
    +-- Indexer Node
    |     +-- Download video with yt-dlp
    |     +-- Upload video to Azure Video Indexer
    |     +-- Extract transcript and OCR text
    |
    +-- Auditor Node
          +-- Embed transcript and OCR text
          +-- Retrieve compliance rules from Azure AI Search
          +-- Analyze content with Azure OpenAI
          +-- Return violations, status, and report
```

## Features

- YouTube video download and Azure Video Indexer processing
- Speech-to-text transcript extraction
- On-screen text extraction through OCR
- Azure AI Search retrieval for regulatory and brand guidance
- Azure OpenAI compliance auditing
- LangGraph multi-node workflow
- FastAPI HTTP endpoint
- CLI workflow runner
- Optional Azure Monitor telemetry

## Technology

- Python 3.12+
- FastAPI and Uvicorn
- LangGraph and LangChain
- Azure OpenAI
- Azure AI Search
- Azure Video Indexer
- Azure Blob, Monitor, and Identity SDKs
- yt-dlp
- PyPDF

## Project Structure

```text
.
├── main.py                         # CLI workflow entry point
├── pyproject.toml                  # Python dependencies and project metadata
├── backend/
│   ├── data/                       # PDF compliance documents
│   ├── scripts/
│   │   └── index_documents.py      # Index PDF documents into Azure AI Search
│   ├── src/
│   │   ├── api/
│   │   │   ├── server.py           # FastAPI application
│   │   │   └── telemetry.py        # Azure Monitor setup
│   │   ├── graph/
│   │   │   ├── workflow.py         # LangGraph workflow definition
│   │   │   ├── nodes.py            # Indexer and auditor nodes
│   │   │   └── state.py            # Workflow state schema
│   │   └── services/
│   │       └── video_indexer.py    # Azure Video Indexer client
│   └── Dockerfile
├── azure_functions/                # Azure Functions deployment files
└── uv.lock                         # Locked dependency versions
```

## Prerequisites

- Python 3.12 or newer
- Azure OpenAI resource with chat and embedding deployments
- Azure AI Search service
- Azure Video Indexer account
- Azure authentication through Azure CLI or another supported DefaultAzureCredential source
- PDF compliance documents placed in `backend/data/`
- FFmpeg may be required by yt-dlp depending on the selected video format

## Setup

### 1. Create and activate a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 2. Install the project

```bash
python -m pip install --upgrade pip
python -m pip install -e .
```

If `uv` is installed, dependencies can also be synchronized with:

```bash
uv sync
```

### 3. Configure environment variables

Create a `.env` file in the project root:

```dotenv
# Azure OpenAI
AZURE_OPENAI_ENDPOINT=https://<resource-name>.openai.azure.com/
AZURE_OPENAI_API_KEY=<your-api-key>
AZURE_OPENAI_API_VERSION=<api-version>
AZURE_OPENAI_CHAT_DEPLOYMENT=<chat-deployment-name>
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small

# Azure AI Search
AZURE_SEARCH_ENDPOINT=https://<search-service-name>.search.windows.net
AZURE_SEARCH_API_KEY=<your-search-admin-key>
AZURE_SEARCH_INDEX_NAME=<index-name>

# Azure Video Indexer
AZURE_VI_ACCOUNT_ID=<video-indexer-account-id>
AZURE_VI_LOCATION=<video-indexer-location>
AZURE_VI_NAME=<video-indexer-resource-name>
AZURE_SUBSCRIPTION_ID=<azure-subscription-id>
AZURE_RESOURCE_GROUP=<azure-resource-group>

# Optional
APPLICATIONINSIGHTS_CONNECTION_STRING=<application-insights-connection-string>
```

Never commit `.env` or API keys to source control.

### 4. Authenticate to Azure

When using Azure CLI authentication:

```bash
az login
```

The Video Indexer service uses `DefaultAzureCredential`, so Azure CLI, managed identity, environment credentials, or another supported credential source may be used.

### 5. Index compliance documents

Add PDF documents to `backend/data/`, then run:

```bash
python backend/scripts/index_documents.py
```

The script creates text chunks and uploads embeddings to the configured Azure AI Search index.

## Run the CLI

```bash
python main.py
```

The default CLI request is a YouTube URL defined in `main.py`. Update `video_url` in `run_cli_simulation()` to audit a different video.

## Run the API

```bash
uvicorn backend.src.api.server:app --reload
```

Open:

- Swagger UI: http://localhost:8000/docs
- Health check: http://localhost:8000/health
- Audit endpoint: http://localhost:8000/audit

### Audit request

```bash
curl -X POST "http://localhost:8000/audit" \
  -H "Content-Type: application/json" \
  -d '{"video_url":"https://youtu.be/dT7S75eYhcQ"}'
```

Example response:

```json
{
  "session_id": "ce6c43bb-c71a-4f16-a377-8b493502fee2",
  "video_id": "vid_ce6c43bb",
  "status": "FAIL",
  "final_report": "Summary of the compliance findings.",
  "compliance_results": [
    {
      "category": "Claim Validation",
      "severity": "CRITICAL",
      "description": "Explanation of the violation"
    }
  ]
}
```

## Docker

Build and run the backend container:

```bash
docker build -t brand-guardian-backend ./backend
docker run --env-file .env -p 8000:8000 brand-guardian-backend
```

The Dockerfile must contain a valid container entry point before the image can be started in this repository.

## Verification

Compile and import-check the project:

```bash
python -m compileall -q main.py backend azure_functions
python -c "from backend.src.graph.workflow import app; print('workflow import OK')"
```

## Current Limitations

- The indexer currently accepts YouTube URLs only.
- Video processing is synchronous and polls Azure Video Indexer every 30 seconds.
- The CLI and API are in-memory; audit results are not persisted to a database.
- Azure Functions and Docker configuration should be completed before deployment.

## Security

- Store credentials in `.env` for local development or Azure Key Vault/managed identity for deployment.
- Restrict Azure AI Search and Azure OpenAI keys to the minimum required scope.
- Do not log API keys or secrets.
- Review licensing, privacy, and data-resident requirements before processing third-party videos.

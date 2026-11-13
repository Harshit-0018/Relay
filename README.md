# The Relay — Phase 1

Phase 1 implements the paper-library foundation: exact-version downloads, page-aware PDF extraction, chunking, BM25 retrieval, a callable retrieval tool, a small CLI, and an initial evaluation-question dataset.

## Requirements

- Python 3.11 or newer
- Internet access for the initial arXiv downloads
- Windows PowerShell instructions are provided below

## Windows PowerShell setup

```powershell
# Extract the ZIP, then open a PowerShell terminal in the `the-relay` folder.
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

If PowerShell blocks activation, run this once in the current terminal and activate again:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

## Download the exact paper versions

```powershell
python scripts/download_papers.py
```

The script downloads only the versions pinned in `data/papers/paper_registry.json`, checks that each response is a PDF, and writes SHA-256 values to `data/papers/download_manifest.json`. It makes up to three attempts per file.

Pinned files:

- `1706.03762v7.pdf` — Attention Is All You Need
- `2410.00037v1.pdf` — Moshi: a speech-text foundation model for real-time dialogue
- `2507.07955v2.pdf` — Dynamic Chunking for End-to-End Hierarchical Sequence Modeling

## Extract and chunk the papers

```powershell
python -m app.cli ingest
```

Optional chunk settings (characters, not tokens):

```powershell
python -m app.cli ingest --chunk-size 1200 --overlap 180
```

Outputs:

- `data/processed/pages.jsonl` — page-level extracted text and metadata
- `data/processed/chunks.jsonl` — retrieval chunks with paper/version/page/section metadata
- `data/processed/ingestion_manifest.json` — counts and active chunk settings

Citation page numbers are 1-based physical PDF page numbers. Printed page labels are not yet reliably detected.

## Search the library

```powershell
python -m app.cli search "multi-head attention and positional encoding" --top-k 5
```

The callable tool interface is available as `app.retrieval.tool.PaperSearchTool`. It returns passages with source metadata and a citation-format hint.

## Tests

```powershell
python -m pytest
```

The test suite uses a generated tiny PDF fixture and does not require network access or a Groq API key.

## Current scope

Phase 1 only. Groq calls, FastAPI, generation, verification, simulation, load testing and final report generation are scheduled for later phases.

## Paper version sources

- https://arxiv.org/pdf/1706.03762v7
- https://arxiv.org/pdf/2410.00037v1
- https://arxiv.org/pdf/2507.07955v2



# The Relay Phase 2

The Relay is a multi-step, evidence-grounded question-answering service over the three version-pinned research papers in the assignment. This ZIP cumulatively contains Phase 1 and Phase 2.

## Requirements

- Python 3.11 or newer
- Internet access only for the first exact-version arXiv downloads and real Groq calls
- Windows PowerShell instructions below

## Windows PowerShell setup

```powershell
# Extract the ZIP and open a terminal in the `the-relay` folder.
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
Copy-Item .env.example .env
```

If PowerShell blocks activation:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

## Download the exact paper versions

```powershell
python scripts/download_papers.py
```

The script downloads only the pinned versions in `data/papers/paper_registry.json`, validates the PDF signature, and writes SHA-256 values to `data/papers/download_manifest.json`.

- `1706.03762v7.pdf` — Attention Is All You Need
- `2410.00037v1.pdf` — Moshi: a speech-text foundation model for real-time dialogue
- `2507.07955v2.pdf` — Dynamic Chunking for End-to-End Hierarchical Sequence Modeling

## Ingest and index the papers

```powershell
python -m app.cli ingest
```

Optional settings are character counts, not token counts:

```powershell
python -m app.cli ingest --chunk-size 1200 --overlap 180
```

Outputs:

- `data/processed/pages.jsonl` — extracted page text and metadata
- `data/processed/chunks.jsonl` — page-bounded retrieval chunks
- `data/processed/ingestion_manifest.json` — ingestion parameters and counts

Citation page numbers currently use 1-based physical PDF page numbers. Printed page labels are not reliably parsed.

## Run the API in offline demo mode

The default `.env.example` uses `PROVIDER_MODE=demo`. Demo mode does not call an LLM and needs no API key. It verifies that retrieval and citation wiring work; its response is clearly labeled as a retrieval smoke test, not as a substantive model-generated answer.

```powershell
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Check service and paper-index readiness:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/health
```

Ask a question:

```powershell
$body = @{ question = "What does the Transformer rely on?"; top_k = 5 } | ConvertTo-Json
Invoke-RestMethod -Uri http://127.0.0.1:8000/v1/ask -Method Post -ContentType "application/json" -Body $body
```

Streaming endpoint (server-sent events):

```powershell
$body = @{ question = "What does the Transformer rely on?"; top_k = 5 } | ConvertTo-Json
curl.exe -N -X POST http://127.0.0.1:8000/v1/ask/stream -H "Content-Type: application/json" -d $body
```

The service generates and validates the whole answer before releasing any text. It then streams the verified text in SSE chunks. This honors the assignment's check-before-send requirement; the first client chunk therefore arrives after generation and citation verification.

API documentation is available at `http://127.0.0.1:8000/docs` while the server is running.

## Enable real Groq generation

Edit `.env`:

```dotenv
PROVIDER_MODE=groq
GROQ_API_KEY=your_key_here
GROQ_MODEL=openai/gpt-oss-20b
```

Restart the server. Do not commit `.env` or share the key. The provider uses Groq's OpenAI-compatible Chat Completions endpoint and supports both non-streaming completions and upstream SSE generation. The default model ID is configurable; check Groq's current model catalogue before changing it.

If `GROQ_API_KEY` is absent while `PROVIDER_MODE=groq`, startup is not blocked, but the first answering request returns a clear configuration error. Switch to `PROVIDER_MODE=demo` for offline testing.

## Search without the API

```powershell
python -m app.cli search "multi-head attention and positional encoding" --top-k 5
```

The callable retrieval tool is `app.retrieval.tool.PaperSearchTool` and is invoked directly by the answer pipeline.

## Tests

```powershell
python -m pytest -q
python -m compileall -q app scripts tests
```

The automated tests use temporary synthetic chunks/PDFs and do not require paper downloads, a Groq key, or external network access.

## Phase 2 pipeline

1. Validate and classify the incoming request through API schema constraints.
2. Invoke the paper retrieval tool.
3. Construct a bounded evidence context with exact paper/version/page labels.
4. Generate through the selected provider (Groq or offline demo).
5. Validate citation provenance against the exact retrieved citation labels.
6. If citation validation fails, perform at most one correction attempt by default.
7. If validation still fails, withhold the draft and return a refusal.

The citation verifier checks that citations match retrieved source references. It does not claim to prove semantic entailment of every generated sentence; answer-quality evaluation is part of the later experiment phase.

## Known Phase 2 limits

- Original arXiv PDFs are not included in this ZIP. Download them locally and run ingestion before asking questions.
- Demo mode is a plumbing/retrieval check, not an LLM substitute.
- The API currently uses one process-local BM25 index. Queueing, concurrency policies, retry/backoff and configurable load simulation are Phase 3.
- Citation validation is provenance validation, not a full semantic fact-checker.
- For streaming, upstream model tokens are buffered until citation validation completes, then delivered as SSE chunks.

## Project layout

```text
the-relay/
├── app/
│   ├── api.py, api_schemas.py, config.py, main.py, cli.py
│   ├── providers/       # Groq and offline demo adapters
│   ├── services/        # pipeline, prompt/context, citation validation
│   └── retrieval/       # ingestion, chunking, BM25 and tool interface
├── data/
│   ├── papers/          # pinned registry; PDFs downloaded locally
│   ├── processed/       # generated pages/chunks
│   └── evaluation_questions/
├── scripts/
├── tests/
├── docs/
├── .env.example
├── requirements.txt
└── PROJECT_STATE.md
```

## Paper version sources

- https://arxiv.org/pdf/1706.03762v7
- https://arxiv.org/pdf/2410.00037v1
- https://arxiv.org/pdf/2507.07955v2



# The Relay Phase 3

The Relay is a multi-step, evidence-grounded question-answering service over the three version-pinned research papers in the assignment. This ZIP cumulatively contains Phases 1, 2 and 3.

## Requirements

- Python 3.11 or newer
- Internet access only for the first exact-version arXiv downloads and real Groq calls
- Windows PowerShell instructions below

## Windows PowerShell setup

```powershell
# Extract the ZIP and open a terminal in the `the-relay` folder.
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
Copy-Item .env.example .env
```

If PowerShell blocks activation:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

## Download the exact paper versions

```powershell
python scripts/download_papers.py
```

The script downloads only the pinned versions in `data/papers/paper_registry.json`, validates the PDF signature, and writes SHA-256 values to `data/papers/download_manifest.json`.

- `1706.03762v7.pdf` — Attention Is All You Need
- `2410.00037v1.pdf` — Moshi: a speech-text foundation model for real-time dialogue
- `2507.07955v2.pdf` — Dynamic Chunking for End-to-End Hierarchical Sequence Modeling

## Ingest and index the papers

```powershell
python -m app.cli ingest
```

Optional settings are character counts, not token counts:

```powershell
python -m app.cli ingest --chunk-size 1200 --overlap 180
```

Outputs:

- `data/processed/pages.jsonl` — extracted page text and metadata
- `data/processed/chunks.jsonl` — page-bounded retrieval chunks
- `data/processed/ingestion_manifest.json` — ingestion parameters and counts

Citation page numbers currently use 1-based physical PDF page numbers. Printed page labels are not reliably parsed.

## Run the API in offline demo mode

The default `.env.example` uses `PROVIDER_MODE=demo`. Demo mode does not call an LLM and needs no API key. It verifies that retrieval and citation wiring work; its response is clearly labeled as a retrieval smoke test, not as a substantive model-generated answer.

```powershell
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Check service and paper-index readiness:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/health
```

Ask a question:

```powershell
$body = @{ question = "What does the Transformer rely on?"; top_k = 5 } | ConvertTo-Json
Invoke-RestMethod -Uri http://127.0.0.1:8000/v1/ask -Method Post -ContentType "application/json" -Body $body
```

Streaming endpoint (server-sent events):

```powershell
$body = @{ question = "What does the Transformer rely on?"; top_k = 5 } | ConvertTo-Json
curl.exe -N -X POST http://127.0.0.1:8000/v1/ask/stream -H "Content-Type: application/json" -d $body
```

The service generates and validates the whole answer before releasing any text. It then streams the verified text in SSE chunks. This honors the assignment's check-before-send requirement; the first client chunk therefore arrives after generation and citation verification.

API documentation is available at `http://127.0.0.1:8000/docs` while the server is running.

## Enable real Groq generation

Edit `.env`:

```dotenv
PROVIDER_MODE=groq
GROQ_API_KEY=your_key_here
GROQ_MODEL=openai/gpt-oss-20b
```

Restart the server. Do not commit `.env` or share the key. The provider uses Groq's OpenAI-compatible Chat Completions endpoint and supports both non-streaming completions and upstream SSE generation. The default model ID is configurable; check Groq's current model catalogue before changing it.

If `GROQ_API_KEY` is absent while `PROVIDER_MODE=groq`, startup is not blocked, but the first answering request returns a clear configuration error. Switch to `PROVIDER_MODE=demo` for offline testing.

## Search without the API

```powershell
python -m app.cli search "multi-head attention and positional encoding" --top-k 5
```

The callable retrieval tool is `app.retrieval.tool.PaperSearchTool` and is invoked directly by the answer pipeline.

## Tests

```powershell
python -m pytest -q
python -m compileall -q app scripts tests
```

The automated tests use temporary synthetic chunks/PDFs and do not require paper downloads, a Groq key, or external network access.

## Phase 2 pipeline

1. Validate and classify the incoming request through API schema constraints.
2. Invoke the paper retrieval tool.
3. Construct a bounded evidence context with exact paper/version/page labels.
4. Generate through the selected provider (Groq or offline demo).
5. Validate citation provenance against the exact retrieved citation labels.
6. If citation validation fails, perform at most one correction attempt by default.
7. If validation still fails, withhold the draft and return a refusal.

The citation verifier checks that citations match retrieved source references. It does not claim to prove semantic entailment of every generated sentence; answer-quality evaluation is part of the later experiment phase.

## Known limitations

- Original arXiv PDFs are not included in this ZIP. Download them locally and run ingestion before asking questions.
- Demo mode is a plumbing/retrieval check, not an LLM substitute.
- The API uses one process-local BM25 index. Queueing, concurrency controls, retry/backoff and configurable load simulation are implemented in Phase 3; these controls are process-local.
- Citation validation is provenance validation, not a full semantic fact-checker.
- For streaming, upstream model tokens are buffered until citation validation completes, then delivered as SSE chunks.

## Project layout

```text
the-relay/
├── app/
│   ├── api.py, api_schemas.py, config.py, main.py, cli.py
│   ├── providers/       # Groq, offline demo and stand-in adapters
│   ├── reliability/    # bounded admission, deadlines, retries, circuit breaker
│   ├── services/        # pipeline, factory and citation validation
│   └── retrieval/       # ingestion, chunking, BM25 and tool interface
├── simulator/           # configurable local model service and standalone API
├── data/
│   ├── papers/          # pinned registry; PDFs downloaded locally
│   ├── processed/       # generated pages/chunks
│   └── evaluation_questions/
├── scripts/
├── tests/
├── docs/
├── .env.example
├── requirements.txt
└── PROJECT_STATE.md
```

## Paper version sources

- https://arxiv.org/pdf/1706.03762v7
- https://arxiv.org/pdf/2410.00037v1
- https://arxiv.org/pdf/2507.07955v2

## Phase 3 reliability and stand-in service

Phase 3 adds a provider reliability wrapper and a configurable local simulator.

### Run the integrated application with the stand-in

In `.env`, set:

```dotenv
PROVIDER_MODE=standin
```

Then run the normal application:

```powershell
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

The stand-in is an in-process provider and requires no Groq key. It returns an explicitly labelled deterministic simulation response; it is for reliability/load testing, not answer-quality grading.

### Run the stand-in as a separate local service

The same simulation engine is also exposed through an OpenAI-compatible local API:

```powershell
uvicorn simulator.api:app --host 127.0.0.1 --port 8010
```

- `GET http://127.0.0.1:8010/health`
- `POST http://127.0.0.1:8010/v1/chat/completions`

This standalone endpoint is useful for direct simulator checks. The integrated application uses the same engine in-process when `PROVIDER_MODE=standin`.

### Reliability controls

Application controls are configured through `.env`:

| Setting | Default | Purpose |
|---|---:|---|
| `MAX_IN_FLIGHT` | 8 | Maximum concurrent provider calls |
| `MAX_QUEUE` | 32 | Maximum additional waiting requests |
| `QUEUE_WAIT_SECONDS` | 10 | Maximum time waiting for a provider slot |
| `PROVIDER_DEADLINE_SECONDS` | 50 | Overall provider-operation deadline, including retries |
| `MAX_PROVIDER_RETRIES` | 2 | Maximum retries after the first attempt |
| `RETRY_BASE_SECONDS` | 0.25 | Initial exponential backoff |
| `RETRY_MAX_SECONDS` | 2 | Maximum locally calculated backoff |
| `RETRY_JITTER` | 0.20 | Added random jitter in seconds |
| `CIRCUIT_FAILURE_THRESHOLD` | 5 | Consecutive server-side failures before opening circuit |
| `CIRCUIT_RECOVERY_SECONDS` | 10 | Time before one half-open recovery probe |

Retry-After is honored as received. If it does not fit inside the overall operation deadline, the current rate-limit error is returned rather than retrying too early.

### Stand-in controls

The simulator reads the `STANDIN_*` variables in `.env.example`:

- Latency: `STANDIN_BASE_LATENCY_SECONDS`, `STANDIN_LATENCY_JITTER_SECONDS`
- Slow tail: `STANDIN_SLOW_TAIL_PROBABILITY`, `STANDIN_SLOW_TAIL_MULTIPLIER`
- Failures: `STANDIN_FAILURE_PROBABILITY`, `STANDIN_FAIL_FIRST_N`
- Timeouts: `STANDIN_TIMEOUT_PROBABILITY`, `STANDIN_TIMEOUT_EVERY_N`, `STANDIN_TIMEOUT_SECONDS`
- Quotas: `STANDIN_RPM_LIMIT`, `STANDIN_TPM_LIMIT`, `STANDIN_REQUESTS_PER_DAY`, `STANDIN_TOKENS_PER_DAY`
- Daily-window duration for shorter reproducible simulations: `STANDIN_DAILY_WINDOW_SECONDS`
- Concurrency/overload: `STANDIN_MAX_CONCURRENT`, `STANDIN_OVERLOAD_THRESHOLD`, `STANDIN_OVERLOAD_SLOWDOWN_FACTOR`
- Streaming: `STANDIN_STREAM_CHUNK_CHARS`, `STANDIN_STREAM_CHUNK_INTERVAL_SECONDS`
- Reproducibility: `STANDIN_RANDOM_SEED`

Set a quota to `0` to disable that quota. The simulator uses fixed windows: a 60-second RPM/TPM window and a daily window measured from simulator start (configurable for tests). This is a deliberate approximation, not a claim to reproduce the provider's internal distributed limiter. Token quota admission reserves prompt tokens plus a configurable completion-token allowance; observed usage is recorded after each call.

Example local stress configuration:

```dotenv
PROVIDER_MODE=standin
STANDIN_BASE_LATENCY_SECONDS=0.15
STANDIN_FAILURE_PROBABILITY=0.10
STANDIN_RPM_LIMIT=12
STANDIN_TPM_LIMIT=5000
STANDIN_REQUESTS_PER_DAY=80
STANDIN_TOKENS_PER_DAY=20000
STANDIN_MAX_CONCURRENT=3
MAX_IN_FLIGHT=3
MAX_QUEUE=6
MAX_PROVIDER_RETRIES=1
```

### Reliability metrics and retry-storm demonstration

Once the API pipeline is initialized, inspect:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/internal/reliability
```

Run the deterministic retry amplification example:

```powershell
python scripts/retry_storm_demo.py
```

It creates a small two-request RPM budget. A transient failure followed by an automatic retry consumes the remaining request slot, causing the next request to be rate-limited. This demonstrates why retries need a budget and why daily exhaustion must never be retried.

### Phase 3 tests

```powershell
python -m pytest -q
python -m compileall -q app simulator scripts tests
```

The reliability layer is process-local. It is suitable for this single-instance assignment implementation, not shared quota coordination across multiple application workers or hosts. Strict per-user fairness, persistent quota state, distributed circuit breaking and adaptive concurrency are deliberately out of scope.


# The Relay

The Relay is a multi-step, evidence-grounded question-answering service over the three version-pinned research papers in the assignment. This ZIP cumulatively contains Phases 1, 2 and 3.

## Requirements

- Python 3.11 or newer
- Internet access only for the first exact-version arXiv downloads and real Groq calls
- Windows PowerShell instructions below

## Windows PowerShell setup

```powershell
# Extract the ZIP and open a terminal in the `the-relay` folder.
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
Copy-Item .env.example .env
```

If PowerShell blocks activation:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

## Download the exact paper versions

```powershell
python scripts/download_papers.py
```

The script downloads only the pinned versions in `data/papers/paper_registry.json`, validates the PDF signature, and writes SHA-256 values to `data/papers/download_manifest.json`.

- `1706.03762v7.pdf` — Attention Is All You Need
- `2410.00037v1.pdf` — Moshi: a speech-text foundation model for real-time dialogue
- `2507.07955v2.pdf` — Dynamic Chunking for End-to-End Hierarchical Sequence Modeling

## Ingest and index the papers

```powershell
python -m app.cli ingest
```

Optional settings are character counts, not token counts:

```powershell
python -m app.cli ingest --chunk-size 1200 --overlap 180
```

Outputs:

- `data/processed/pages.jsonl` — extracted page text and metadata
- `data/processed/chunks.jsonl` — page-bounded retrieval chunks
- `data/processed/ingestion_manifest.json` — ingestion parameters and counts

Citation page numbers currently use 1-based physical PDF page numbers. Printed page labels are not reliably parsed.

## Run the API in offline demo mode

The default `.env.example` uses `PROVIDER_MODE=demo`. Demo mode does not call an LLM and needs no API key. It verifies that retrieval and citation wiring work; its response is clearly labeled as a retrieval smoke test, not as a substantive model-generated answer.

```powershell
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Check service and paper-index readiness:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/health
```

Ask a question:

```powershell
$body = @{ question = "What does the Transformer rely on?"; top_k = 5 } | ConvertTo-Json
Invoke-RestMethod -Uri http://127.0.0.1:8000/v1/ask -Method Post -ContentType "application/json" -Body $body
```

Streaming endpoint (server-sent events):

```powershell
$body = @{ question = "What does the Transformer rely on?"; top_k = 5 } | ConvertTo-Json
curl.exe -N -X POST http://127.0.0.1:8000/v1/ask/stream -H "Content-Type: application/json" -d $body
```

The service generates and validates the whole answer before releasing any text. It then streams the verified text in SSE chunks. This honors the assignment's check-before-send requirement; the first client chunk therefore arrives after generation and citation verification.

API documentation is available at `http://127.0.0.1:8000/docs` while the server is running.

## Enable real Groq generation

Edit `.env`:

```dotenv
PROVIDER_MODE=groq
GROQ_API_KEY=your_key_here
GROQ_MODEL=openai/gpt-oss-20b
```

Restart the server. Do not commit `.env` or share the key. The provider uses Groq's OpenAI-compatible Chat Completions endpoint and supports both non-streaming completions and upstream SSE generation. The default model ID is configurable; check Groq's current model catalogue before changing it.

If `GROQ_API_KEY` is absent while `PROVIDER_MODE=groq`, startup is not blocked, but the first answering request returns a clear configuration error. Switch to `PROVIDER_MODE=demo` for offline testing.

## Search without the API

```powershell
python -m app.cli search "multi-head attention and positional encoding" --top-k 5
```

The callable retrieval tool is `app.retrieval.tool.PaperSearchTool` and is invoked directly by the answer pipeline.

## Tests

```powershell
python -m pytest -q
python -m compileall -q app scripts tests
```

The automated tests use temporary synthetic chunks/PDFs and do not require paper downloads, a Groq key, or external network access.

## Phase 2 pipeline

1. Validate and classify the incoming request through API schema constraints.
2. Invoke the paper retrieval tool.
3. Construct a bounded evidence context with exact paper/version/page labels.
4. Generate through the selected provider (Groq or offline demo).
5. Validate citation provenance against the exact retrieved citation labels.
6. If citation validation fails, perform at most one correction attempt by default.
7. If validation still fails, withhold the draft and return a refusal.

The citation verifier checks that citations match retrieved source references. It does not claim to prove semantic entailment of every generated sentence; answer-quality evaluation is part of the later experiment phase.

## Known limitations

- Original arXiv PDFs are not included in this ZIP. Download them locally and run ingestion before asking questions.
- Demo mode is a plumbing/retrieval check, not an LLM substitute.
- The API uses one process-local BM25 index. Queueing, concurrency controls, retry/backoff and configurable load simulation are implemented in Phase 3; these controls are process-local.
- Citation validation is provenance validation, not a full semantic fact-checker.
- For streaming, upstream model tokens are buffered until citation validation completes, then delivered as SSE chunks.

## Project layout

```text
the-relay/
├── app/
│   ├── api.py, api_schemas.py, config.py, main.py, cli.py
│   ├── providers/       # Groq, offline demo and stand-in adapters
│   ├── reliability/    # bounded admission, deadlines, retries, circuit breaker
│   ├── services/        # pipeline, factory and citation validation
│   └── retrieval/       # ingestion, chunking, BM25 and tool interface
├── simulator/           # configurable local model service and standalone API
├── data/
│   ├── papers/          # pinned registry; PDFs downloaded locally
│   ├── processed/       # generated pages/chunks
│   └── evaluation_questions/
├── scripts/
├── tests/
├── docs/
├── .env.example
├── requirements.txt
└── PROJECT_STATE.md
```

## Paper version sources

- https://arxiv.org/pdf/1706.03762v7
- https://arxiv.org/pdf/2410.00037v1
- https://arxiv.org/pdf/2507.07955v2

## Phase 3 reliability and stand-in service

Phase 3 adds a provider reliability wrapper and a configurable local simulator.

### Run the integrated application with the stand-in

In `.env`, set:

```dotenv
PROVIDER_MODE=standin
```

Then run the normal application:

```powershell
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

The stand-in is an in-process provider and requires no Groq key. It returns an explicitly labelled deterministic simulation response; it is for reliability/load testing, not answer-quality grading.

### Run the stand-in as a separate local service

The same simulation engine is also exposed through an OpenAI-compatible local API:

```powershell
uvicorn simulator.api:app --host 127.0.0.1 --port 8010
```

- `GET http://127.0.0.1:8010/health`
- `POST http://127.0.0.1:8010/v1/chat/completions`

This standalone endpoint is useful for direct simulator checks. The integrated application uses the same engine in-process when `PROVIDER_MODE=standin`.

### Reliability controls

Application controls are configured through `.env`:

| Setting | Default | Purpose |
|---|---:|---|
| `MAX_IN_FLIGHT` | 8 | Maximum concurrent provider calls |
| `MAX_QUEUE` | 32 | Maximum additional waiting requests |
| `QUEUE_WAIT_SECONDS` | 10 | Maximum time waiting for a provider slot |
| `PROVIDER_DEADLINE_SECONDS` | 50 | Overall provider-operation deadline, including retries |
| `MAX_PROVIDER_RETRIES` | 2 | Maximum retries after the first attempt |
| `RETRY_BASE_SECONDS` | 0.25 | Initial exponential backoff |
| `RETRY_MAX_SECONDS` | 2 | Maximum locally calculated backoff |
| `RETRY_JITTER` | 0.20 | Added random jitter in seconds |
| `CIRCUIT_FAILURE_THRESHOLD` | 5 | Consecutive server-side failures before opening circuit |
| `CIRCUIT_RECOVERY_SECONDS` | 10 | Time before one half-open recovery probe |

Retry-After is honored as received. If it does not fit inside the overall operation deadline, the current rate-limit error is returned rather than retrying too early.

### Stand-in controls

The simulator reads the `STANDIN_*` variables in `.env.example`:

- Latency: `STANDIN_BASE_LATENCY_SECONDS`, `STANDIN_LATENCY_JITTER_SECONDS`
- Slow tail: `STANDIN_SLOW_TAIL_PROBABILITY`, `STANDIN_SLOW_TAIL_MULTIPLIER`
- Failures: `STANDIN_FAILURE_PROBABILITY`, `STANDIN_FAIL_FIRST_N`
- Timeouts: `STANDIN_TIMEOUT_PROBABILITY`, `STANDIN_TIMEOUT_EVERY_N`, `STANDIN_TIMEOUT_SECONDS`
- Quotas: `STANDIN_RPM_LIMIT`, `STANDIN_TPM_LIMIT`, `STANDIN_REQUESTS_PER_DAY`, `STANDIN_TOKENS_PER_DAY`
- Daily-window duration for shorter reproducible simulations: `STANDIN_DAILY_WINDOW_SECONDS`
- Concurrency/overload: `STANDIN_MAX_CONCURRENT`, `STANDIN_OVERLOAD_THRESHOLD`, `STANDIN_OVERLOAD_SLOWDOWN_FACTOR`
- Streaming: `STANDIN_STREAM_CHUNK_CHARS`, `STANDIN_STREAM_CHUNK_INTERVAL_SECONDS`
- Reproducibility: `STANDIN_RANDOM_SEED`

Set a quota to `0` to disable that quota. The simulator uses fixed windows: a 60-second RPM/TPM window and a daily window measured from simulator start (configurable for tests). This is a deliberate approximation, not a claim to reproduce the provider's internal distributed limiter. Token quota admission reserves prompt tokens plus a configurable completion-token allowance; observed usage is recorded after each call.

Example local stress configuration:

```dotenv
PROVIDER_MODE=standin
STANDIN_BASE_LATENCY_SECONDS=0.15
STANDIN_FAILURE_PROBABILITY=0.10
STANDIN_RPM_LIMIT=12
STANDIN_TPM_LIMIT=5000
STANDIN_REQUESTS_PER_DAY=80
STANDIN_TOKENS_PER_DAY=20000
STANDIN_MAX_CONCURRENT=3
MAX_IN_FLIGHT=3
MAX_QUEUE=6
MAX_PROVIDER_RETRIES=1
```

### Reliability metrics and retry-storm demonstration

Once the API pipeline is initialized, inspect:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/internal/reliability
```

Run the deterministic retry amplification example:

```powershell
python scripts/retry_storm_demo.py
```

It creates a small two-request RPM budget. A transient failure followed by an automatic retry consumes the remaining request slot, causing the next request to be rate-limited. This demonstrates why retries need a budget and why daily exhaustion must never be retried.

### Phase 3 tests

```powershell
python -m pytest -q
python -m compileall -q app simulator scripts tests
```

The reliability layer is process-local. It is suitable for this single-instance assignment implementation, not shared quota coordination across multiple application workers or hosts. Strict per-user fairness, persistent quota state, distributed circuit breaking and adaptive concurrency are deliberately out of scope.

## Phase 4 — Benchmarking and results

Phase 4 adds five reproducible stand-in load scenarios, three retrieval configurations, request-level CSV logging, JSONL timing traces, machine-readable metrics and automatically generated assignment tables.

### Run the stand-in load scenarios

```powershell
python -m experiments.run_load_benchmark
```

This runs steady traffic, a sudden burst, sustained overload, a halved per-minute limit and daily-budget exhaustion. It writes the exact assignment schema to project-root `results.csv` and detailed outputs under `experiments/results/`. Re-running the load benchmark replaces the prior stand-in rows while preserving real rows already collected.

### Run retrieval comparisons

First download and ingest the exact paper versions:

```powershell
python scripts/download_papers.py
python -m app.cli ingest
```

Then compare three chunking/top-k/reranking configurations:

```powershell
python -m experiments.retrieval_benchmark
```

This computes annotated evidence-hit diagnostics from the 20-question dataset. Evidence-hit rate is retrieval recall against expected paper/page/term annotations; it is not a final answer correctness score.

### Optional real Groq baseline

The real probe sends **no requests by default**. Configure your own key in `.env`, check your organization's Limits page, then explicitly authorize at most five small requests:

```powershell
python -m experiments.run_real_groq_probe --run --max-requests 3
```

The probe records TTFT, total latency, HTTP status, token usage when returned, and available rate-limit headers. It does not intentionally exhaust any quota.

For actual answer-quality evaluation across all three retrieval settings (up to 15 questions per setting, 45 total):

```powershell
python -m experiments.retrieval_benchmark --with-groq --max-questions 15
```

This requires the original PDFs and `GROQ_API_KEY`, and may consume provider quota. It uses sequential requests and conservative pacing. Check current account limits before running it. Grades use an inspectable expected-term plus exact paper/page citation heuristic; manually review labels before submission.

### Rebuild assignment tables

```powershell
python -m experiments.build_report_tables
```

Outputs include:

- `results.csv` — one row per request, exact required columns.
- `experiments/results/traces.jsonl` — queue wait, work, retry wait, TTFT and post-first-token timing.
- `experiments/results/scenario_metrics.json` and `metrics.json` — measured scenario and aggregate metrics.
- `experiments/results/retrieval_settings.csv` and `retrieval_question_results.csv` — retrieval settings and evidence-hit diagnostic.
- `experiments/results/table_1_real_service_limits.csv`, `table_2a_latency.csv`, `table_2b_outcomes.csv`, `table_3_retrieval.csv`.
- `experiments/results/report_tables.md` — tables in report-ready Markdown.
- `experiments/results/slow_request_analysis.json` — measured breakdown for the slowest successful stand-in request.
- `experiments/results/real_probe_status.json` — whether a real probe ran or was blocked.

See `docs/PHASE4.md` and `docs/GROQ_LIMITS_SNAPSHOT.md` for the methodology, limit snapshot and boundaries.


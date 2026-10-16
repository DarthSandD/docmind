# DocMind

Retrieval-augmented question answering over a document set you control.

**Live:** https://darthsandd.github.io/docmind/
**Demo application:** http://129.225.4.69

DocMind indexes documents, retrieves the passages that actually bear on a question,
and writes its answer from those passages alone. Every claim carries a numbered
citation back to the passage it came from, and the retrieved passages stay visible
underneath the answer so any statement can be checked against its source.

## How it works

1. **Index** — documents are split into ~900-character passages with 150 characters of
   overlap, embedded into vectors, and stored in a persistent ChromaDB collection.
2. **Retrieve** — the question is embedded with the same model and compared against
   every stored vector by cosine distance; the five closest passages are returned with
   their similarity scores.
3. **Ground** — the retrieved passages are numbered and placed in the prompt as the only
   permitted evidence, with an explicit instruction to cite each claim and to say when
   the answer is absent.
4. **Answer** — the model streams its answer token by token; citation markers link back
   to the passage cards rendered beneath it.

## Stack

- **Retrieval:** ChromaDB, cosine distance, top-k = 5
- **Embeddings:** `all-MiniLM-L6-v2` via ONNX, running locally — no embedding API
- **Generation:** Qwen2.5 3B served by Ollama on the same host, streamed over SSE
- **Service:** FastAPI on Uvicorn under systemd, with an ingest API for URLs, pasted
  text and PDF uploads
- **Hardware:** 4 CPU cores, 24 GB memory, no GPU

## Running it

```bash
python3 -m venv venv && ./venv/bin/pip install \
  chromadb fastapi "uvicorn[standard]" httpx beautifulsoup4 pypdf python-multipart
ollama pull qwen2.5:3b

export DOCMIND_MODEL=qwen2.5:3b
export DOCMIND_INGEST_TOKEN=$(openssl rand -hex 16)
./venv/bin/uvicorn app:app --host 0.0.0.0 --port 80
```

## Ingestion is operator-only

Writing to the index and fetching remote pages require an `X-Ingest-Token` header.
Every URL passed to the ingest endpoint is resolved and checked against private,
loopback, link-local, reserved and multicast address ranges before any request is
made, so the index cannot be turned into a probe of the machine it runs on.

## Limits

A 3B model on CPU is a deliberate trade: strong when the retrieved passages contain
the answer, weaker at synthesis across many passages. Most of the quality lives in the
retrieval step — chunk size, overlap and top-k are the settings worth tuning first.

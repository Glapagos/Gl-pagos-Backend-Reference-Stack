# Glápagos Backend — Reference Stack

> **The platform where your data stays and the Americas move.**  
> A modular, provider-agnostic backend for federated data and agentic AI at production scale.

[![License: MIT](https://img.shields.io/badge/License-MIT-7b75d9.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.12-blue)](https://python.org)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-2.0-green)](https://swagger.io)
[![Built by Glápagos](https://img.shields.io/badge/Built%20by-Gl%C3%A1pagos-7b75d9)](https://www.glapagos.com)

This repo contains the **annotated reference implementation** of the Glápagos backend stack — every layer documented, every config explained, production-hardened. Fork what you need. Star what you find useful.

→ **Platform:** [glapagos.com/platform](https://www.glapagos.com/platform)  
→ **Request access:** [glapagos.com](https://www.glapagos.com)

---

## Stack Overview

```
GENIA-AMERICAS / GLAPAGOS-BACKEND
├── API Layer            REST 3.12 · RS256 JWT · OpenAPI 2.0 · Hierarchical Routing
├── Real-Time Layer      WebSocket · ASGI · Pub/Sub Channel Layer · TLS 1.3 / HTTP2
├── Agentic AI/ML Layer  LLM Inference · RAG Pipelines · Federated Training · Typed Schema Validation
├── Federated Data Infra Object Storage · Columnar Analytics · NoSQL Document Store · RDBMS
├── Security Layer       Argon2id · AES-256 Field Encryption · TOTP 2FA · CORS Policy
└── Deployment           Containerised · WSGI + ASGI · Observability · Static Asset CDN
```

All layers are **independently deployable** and **provider-agnostic**. Swap cloud vendors, LLM providers, or storage backends without touching core logic.

---

## Layers

### 1. API Layer

**REST 3.12 · RS256 JWT · OpenAPI 2.0 · Hierarchical Routing**

FastAPI boilerplate with:
- RS256 JWT auth (asymmetric keys, rotation-safe)
- OpenAPI 2.0 spec generated automatically from route definitions
- Hierarchical route scoping (`/v1/americas/{region}/...`)
- Built-in request ID propagation for distributed tracing

```python
# Typical route pattern
@router.get("/{region}/data", dependencies=[Depends(verify_rs256_jwt)])
async def get_regional_data(region: str, scope: TokenScope = Depends()):
    ...
```

**Why RS256 over HS256?** Asymmetric signatures let downstream services verify tokens without sharing a secret. Critical for multi-cloud, multi-tenant deployments.

---

### 2. Real-Time Layer

**WebSocket · ASGI · Pub/Sub Channel Layer · TLS 1.3 / HTTP2**

Native WebSocket support via ASGI — no separate socket server required:

```python
# Channel layer pattern
async def connect(self):
    await self.channel_layer.group_add(self.room_name, self.channel_name)
    await self.accept()

async def receive(self, text_data):
    await self.channel_layer.group_send(self.room_name, {
        "type": "broadcast.message",
        "payload": json.loads(text_data)
    })
```

- Pub/Sub backed by Redis or in-process for local dev
- TLS 1.3 enforced in staging/prod environments
- HTTP/2 multiplexing reduces connection overhead for high-frequency updates

---

### 3. Agentic AI / ML Layer

**LLM Inference · RAG Pipelines · Federated Training · Typed Schema Validation**

Provider-agnostic LLM interface — swap OpenAI, Anthropic, Cohere, or local models without touching application code:

```python
class LLMProvider(Protocol):
    async def complete(self, prompt: str, schema: BaseModel) -> BaseModel: ...

class RAGPipeline:
    def __init__(self, retriever: Retriever, llm: LLMProvider):
        self.retriever = retriever
        self.llm = llm

    async def query(self, question: str, schema: BaseModel) -> BaseModel:
        context = await self.retriever.fetch(question)
        return await self.llm.complete(
            prompt=build_prompt(question, context),
            schema=schema
        )
```

**Typed Schema Validation** — all LLM outputs validated against Pydantic schemas before they touch application logic. No unstructured strings leaking into your stack.

**Federated Training hooks** — data never leaves its region. Model gradients aggregate across nodes; raw data stays sovereign.

---

### 4. Federated Data Infrastructure

**Object Storage · Columnar Analytics · NoSQL Document Store · RDBMS**

Unified query abstraction over multiple backends:

| Backend | Use Case | Default Provider |
|---|---|---|
| Object Storage | Files, blobs, model artifacts | S3-compatible (GCS / AWS / MinIO) |
| Columnar Analytics | Time-series, event data, ML features | BigQuery / DuckDB local |
| NoSQL Document Store | Flexible schemas, user profiles | Firestore / MongoDB |
| RDBMS | Transactional data, FK relationships | PostgreSQL |

Data sovereignty by design — route queries to the correct regional store via config, not code changes.

---

### 5. Security Layer

**Argon2id · AES-256 Field Encryption · TOTP 2FA · CORS Policy**

```python
# Field-level encryption — encrypt before write, decrypt on read
class EncryptedField:
    def __set__(self, obj, value):
        obj.__dict__[self.name] = aes256_encrypt(value, key=current_key())

    def __get__(self, obj, _):
        return aes256_decrypt(obj.__dict__[self.name], key=current_key())

class UserProfile(BaseModel):
    email: str  # plaintext — used for lookups
    tax_id: EncryptedField  # AES-256 at rest
    phone: EncryptedField
```

- **Argon2id** for password hashing (memory-hard, side-channel resistant)
- **AES-256** field-level encryption — encrypt individual columns, not whole databases
- **TOTP 2FA** — RFC 6238 compliant, works with any authenticator app
- **CORS Policy** — per-environment config, allowlist-based

---

### 6. Deployment

**Containerised · WSGI + ASGI · Observability · Static Asset CDN**

```
Local  →  docker compose up
Staging  →  Cloud Run (preview URLs, ephemeral DBs)
Prod  →  GKE / GCP (regional, auto-scaled, CDN-fronted)
```

Observability built in from day one:
- Structured JSON logs (no print statements)
- OpenTelemetry traces propagated across all layers
- Health check endpoints (`/healthz`, `/readyz`) on every service
- Prometheus metrics exposed at `/metrics`

---

## Quick Start

```bash
git clone https://github.com/glapagos/glapagos-backend
cd glapagos-backend
cp .env.example .env          # fill in your secrets
docker compose up             # all layers, local dev
```

Visit `http://localhost:8000/docs` for the interactive OpenAPI explorer.

---

## Environment Matrix

| Feature | Local | Staging | Prod |
|---|---|---|---|
| TLS | Self-signed | Let's Encrypt | Managed cert |
| Database | SQLite + DuckDB | Cloud SQL | Cloud SQL HA |
| Object Storage | MinIO | GCS bucket | GCS multi-region |
| LLM Provider | Ollama (local) | Anthropic API | Anthropic API |
| Auth | Dev JWT (HS256) | RS256 | RS256 + rotation |
| CDN | None | Cloud CDN (preview) | Cloud CDN (full) |

---

## Why Glápagos?

- **Data sovereignty first.** Built for the Americas — data residency requirements across 35+ countries informed every architectural decision.
- **No lock-in.** Every external dependency is behind an interface. Swap providers in config.
- **Production scale, not toy demos.** This stack runs real workloads. The reference implementation is what we actually deploy.

---

## Contributing

Issues and PRs welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

For larger discussions — new layer proposals, provider integrations, regional compliance questions — open a GitHub Discussion.

---

## License

MIT. Use it, fork it, build on it.

→ [glapagos.com/platform](https://www.glapagos.com/platform) · [Request Access](https://www.glapagos.com)

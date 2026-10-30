# AutoShowroom AI

Car showroom WhatsApp sales agent with a RAG knowledge base. Customers ask about prices, trims, specs, and photos on WhatsApp (text, voice notes, or images), and an AI agent answers from the live catalog. Staff drop files into a Google Drive folder and the knowledge base updates itself.

Built for the Egyptian market (Egyptian Arabic replies, EGP prices, local phone formats).

## Features

**WhatsApp AI agent**
- Answers price, trim, color, spec, and comparison questions from the catalog only (no invented data)
- Understands voice notes (Gemini transcription) and customer photos (car model and text detection)
- Sends car photos automatically after the text reply, taken only from tool results
- Per-customer memory (last 10 messages), model fallback (OpenRouter, then Ollama)
- Hands off to the sales team for booking, test drives, and negotiation

**Knowledge base ingestion**
- Watches a Google Drive folder, picks up new files every minute
- Supports text, CSV, JSON catalog, HTML/XML, PDF, scanned PDF, images, docx, pptx, xlsx, and ZIP archives
- Scanned PDFs and images go through Gemini OCR with visual description
- JSON catalog: one car per chunk with `carId`, `brand`, `model` metadata
- Re-uploading a file replaces its old rows, no duplicates

## Tech stack

| Layer | Tools |
|---|---|
| Automation | n8n (webhooks, AI Agent, LangChain nodes) |
| Database / Storage | Supabase (Postgres, pgvector, Storage, RPC) |
| Embeddings | Ollama `nomic-embed-text` |
| LLM | OpenRouter `gpt-4o-mini` (primary), Ollama `qwen2.5:7b` (fallback) |
| OCR / Voice / Vision | Google Gemini `gemini-3.5-flash-lite` |
| WhatsApp | Evolution API v2 |
| Files | Google Drive (OAuth2) |

## Architecture

```
Google Drive (rag folder)
        │
        ▼
Drive Trigger ─► Download ─► Classify ─► Route Kind
                                │ zip ─► Decompress ─► Classify
                                ├ text ───────► Read Text ──────┐
                                ├ pdf ─► Extract ─► scanned? ─► OCR Loop (Gemini)
                                ├ image ─► OCR Loop + Storage upload
                                └ office ─► Unzip ─► Extract ───┤
                                                                ▼
                                        Chunk ─► Delete old rows ─► Embed ─► Supabase pgvector
                                                                                    ▲
Customer ─► WhatsApp ─► Evolution API ─► n8n Webhook                                │
                                            │                                       │
                              voice/image ─► Gemini ─► text                         │
                                            ▼                                       │
                                        AI Agent ── search_car_catalog ─────────────┘
                                            │    └─ search_car_images (RPC)
                                            ▼
                              Build Messages ─► Evolution sendText / sendMedia
```

## Getting started

### 1. Import

Import the workflow JSON into n8n and re-link every credential.

### 2. Credentials

| Credential | Type | Used by |
|---|---|---|
| Google Drive | `googleDriveOAuth2Api` | Drive Trigger, Download File |
| Supabase | `supabaseApi` | Vector stores, RPC calls, Storage |
| Google Gemini | `googlePalmApi` | OCR, transcription, image analysis |
| OpenRouter | `openRouterApi` | Agent primary model |
| Ollama | `ollamaApi` | Embeddings, fallback model |
| Header Auth | `httpHeaderAuth` | Evolution API. Name `apikey`, value is the key, no `Bearer` |

### 3. Database

In the Supabase SQL Editor create:
- `documents` table with a `vector(768)` column (matches `nomic-embed-text`)
- RPC `match_documents`
- RPC `delete_documents_by_files` (takes `keys: [{driveFileName, fileName}]`)
- RPC `search_car_images` (takes `q`)
- A public Storage bucket for car images

### 4. WhatsApp

1. Run n8n and Evolution API in Docker. Reach Evolution from n8n at `http://host.docker.internal:8080`, or use a shared Docker network and the container name.
2. Create an Evolution instance named `whatsapp_agent`.
3. Set the instance webhook to `POST {N8N_URL}/webhook/whatsapp`, event `MESSAGES_UPSERT`.

### 5. Run

1. Drop catalog files into the Drive `rag` folder.
2. Activate the workflow.
3. Message the WhatsApp number.

## Known issues

| Issue | Fix |
|---|---|
| OCR branch inserts into `car_vector_store`, agent reads `documents`, so OCR text is never searched | Insert OCR chunks into `documents` |
| Upload bucket `car_images` vs reply URL bucket `car-images` | Use one bucket name in both nodes |
| `Is Audio?` uses `notEquals "audio"`, always true for `audio/ogg` | Change to `startsWith "audio"` |
| `Return3` references `Base64 to Binary1`, node name is `Base64 to Binary` | Fix the reference |

## Security notes

- Keep all keys in n8n credentials. Never inline them in nodes or prompts.
- The agent ignores requests to reveal its instructions or change its rules.
- Images are sent only from `search_car_images` results, so the model cannot inject outbound links.
- The webhook has no auth yet. Add a secret header check in the first node.
- Do not commit exported workflow JSON with credential IDs or project URLs to a public repo.

## Project structure

```
workflow.json     exported n8n workflow
README.md
```

## Author

**Faris Ataa Lashen**, AI Automation Engineer (n8n, AI Agents, RAG, LLMs)
Portfolio: https://farislashen7-hub.github.io/portfolio · LinkedIn: https://linkedin.com/in/farislashen

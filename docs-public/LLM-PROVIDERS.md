# Providers de LLM e embeddings

> Como o agente decide "quem escreve a resposta" e onde encaixar um provider novo.
> Backend dos providers: `backend/app/infrastructure/integrations/llm_providers.py`.

## 1. LLM (respostas do agente)

O backend é env-controlado: `LLM_PROVIDER_BACKEND`

| valor | o que faz | quando usar |
|---|---|---|
| `echo` | responde fixo "(echo) recebi sua mensagem…" | dev/CI puro, sem custo |
| `rule` | respostas heurísticas (sem rede) | teste de pipeline em offline |
| `openai` | **POST `{LLM_OPENAI_BASE_URL}/chat/completions`** com Bearer `OPENAI_API_KEY` — qualquer API **OpenAI-compatible** | produção |
| `ollama` | POST `{ollama_base_url}/api/chat` (modelo em `LLM_MODEL`) | local/offline |

### 1.1 Qualquer API OpenAI-compatible serve (openai backend)

Config实录 no `.env` (todos os três abaixo são leitura direta do `Settings`):

```
LLM_PROVIDER_BACKEND=openai
LLM_OPENAI_BASE_URL=https://api.openai.com/v1        # troque pelo gateway que usar
LLM_MODEL=<modelo do catálogo>
OPENAI_API_KEY=<chave>
```

Base URLs conhecidas (todas funcionam sem mudar código):

| Gateway | `LLM_OPENAI_BASE_URL` |
|---|---|
| OpenAI | `https://api.openai.com/v1` |
| Groq | `https://api.groq.com/openai/v1` (ex.: `qwen/qwen3.8-27b`) |
| OpenRouter | `https://openrouter.ai/api/v1` |
| OpenCode Inference | `https://opencode.ai/inference/openai/v1` |
| Azure/OpenAI-proxy próprio | o endpoint `/chat/completions` que expuser |

> `llm_openai_base_url` é um campo **real** de `app/core/config.py`
> (adicionado 2026-10). Se você rodar código antigo sem o campo, o env seria
> ignorado silenciosamente — vale conferir no Settings/`GET /docs` o `/v1` schema.

### 1.2 Contrato de saída (importante)

O agente responde em **JSON estruturado** (`response_format: json_object`):
`{response, handoff, handoff_reason, memory_updates[], intent}` — o pipeline lê
esse JSON para decidir envio/fila de handoff. **Portanto o modelo precisa
suportar JSON mode** e budget de tokens suficiente (`LLM_MAX_OUTPUT_TOKENS`, default 700).

Comportamento observado (out/2026):
- `qwen/qwen3.8-27b` (Groq): JSON mode OK em PT-BR ✅
- `openai/gpt-oss-120b` (Groq): falha JSON mode com budget curto (emite
  raciocínio antes do JSON) ⚠️
- Modelos "free" da OpenCode Inference: **só** funcionam de dentro do app OpenCode
  (`FreeTierError` para API keys) ❌

## 2. Embeddings (RAG)

Separado do LLM: `EMBEDDING_PROVIDER_BACKEND`

| valor | como funciona | quando usar |
|---|---|---|
| `hash` | embedding determinístico local SHA (sem rede) | dev/CI (funciona, similaridade pobre) |
| `openai` | `api.openai.com/v1/embeddings` com `OPENAI_API_KEY` (`OPENAI_EMBEDDING_MODEL`) | produção cloud |
| `ollama` | `{ollama_base_url}/api/embeddings` (`OLLAMA_EMBEDDING_MODEL` p.ex. `nomic-embed-text`) | local |

Dimensão configurável: `EMBEDDING_DIMENSIONS` (default 1024 — MESMA valor precisa
no column vetoring pgvector; mudar exige reingestão dos documentos).

## 3. Como testar após trocar provider

```
1. docker compose up -d --build --force-recreate api worker     (env/code novo)
2. POST /v1/agents/playground {"message": "..."}                (login owner)
   → resposta humana, `llm_provider` no payload, `latency_ms` baixa
3. WhatsApp real: mensagem de teste → log `pipeline.replied engine=openai`
```

Chaves em `backend/.env` — nunca comitar. `docs/SECURITY.md` cobre o resto.

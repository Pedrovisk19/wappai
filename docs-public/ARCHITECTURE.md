# Arquitetura — WAppAI

## 1. Visão geral

Monólito modular multi-tenant em camadas. Cada módulo de negócio (identidade, CRM, conversas, conhecimento, agente, faturamento) vive em um pacote com fronteiras explícitas; comunicação entre camadas por interfaces (ports). O domínio não conhece FastAPI, SQLAlchemy ou provedores externos.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Presentation (FastAPI)  /v1/auth /customers /conversations /whatsapp ... │
│   deps (JWT→AuthContext→tenant) · middleware (correlação, headers)       │
├──────────────────────────────────────────────────────────────────────────┤
│ Application (use cases)  AuthService · MessagePipeline · AgentEngine     │
│   ContextBuilder · RAGService · PolicyEngine · DebounceService           │
├──────────────────────────────────────────────────────────────────────────┤
│ Domain          entidades · enums · portas (repositórios/provedores)     │
├──────────────────────────────────────────────────────────────────────────┤
│ Infrastructure  SQLAlchemy/Alembic · Redis/ARQ · WhatsApp adapter        │
│                 LLM/Embeddings adapter · Storage · E-mail · pgvector     │
└──────────────────────────────────────────────────────────────────────────┘
        │                    │                        │
   PostgreSQL 16       Redis 7 + ARQ          Meta WhatsApp Cloud API
   (pgvector, trgm)    (fila/debounce/cache)  (interface WhatsAppProvider)
```

## 2. Multi-tenancy

- **Tenant = Organization.** Toda entidade de negócio carrega `organization_id` (direto ou via pai).
- **Derivação do tenant:** apenas server-side — do JWT (`org` claim) validado contra membership ativa no banco a cada requisição (`AuthContext`). Nunca do body/query do frontend. Superadmins podem apontar um tenant via header `X-Organization-ID` (validado contra o banco).
- **Aplicação:** repositórios recebem `organization_id` do `AuthContext`; nenhuma query de negócio o omite.
- **Defesa em profundidade (implementada na fase 2):** RLS do PostgreSQL (`ENABLE + FORCE`) com GUCs por transação (`app.current_org`/`app.current_user_id`/`app.bypass_rls`); a aplicação conecta como role restrita `wappai_app` e as migrations como admin. Detalhes e trade-offs em `docs/SECURITY.md`.
- **Testes obrigatórios** (markers `isolation`): A não vê B em API, RAG, arquivos, clientes, conversas, mensagens, métricas, tools, memória — em `tests/integration/test_tenant_isolation.py`.

## 3. Pipeline de mensagem

```
WhatsApp Cloud API → POST /whatsapp/webhook/{org_public_id}
  1 validação de assinatura HMAC (X-Hub-Signature-256, app secret)
  2 registro do evento (unique external_event_id → idempotência)
  3 resposta 200 imediata (nunca bloqueia na IA)
  4 enqueue ARQ (process_message)
Worker:
  5 resolução org → conta → cliente (upsert por phone) → conversa (OPEN/AI_ACTIVE)
  6 persistência da mensagem inbound (unique external_message_id)
  7 debounce (janela configurável por org; agrupa mensagens)
  8 ContextBuilder: últimas N mensagens + resumo + memórias + dados do cliente
  9 RAG híbrido (pgvector semântico + tsvector keyword + filtros metadata + rerank leve)
 10 BusinessRules/PolicyEngine → AgentEngine (LLM provider-agnóstico, tools)
 11 validação anti-alucinação da resposta → persistência → envio via WhatsAppProvider
 12 analytics/usage/llm_usage + audit
```

`request_id/correlation_id` atravessa tudo (`core/context.py`, middleware, logs `log_event`).

## 4. Modelo de dados (resumo)

Identidade/acesso: `organizations`, `organization_users`, `organization_settings`, `users`, `refresh_sessions`, `password_reset_tokens`, `audit_logs`.
WhatsApp: `whatsapp_accounts`, `whatsapp_webhook_events`.
CRM/conversas: `customers`, `conversations`, `messages` (idempotência por `external_message_id`), `conversation_summaries`, `customer_memories`.
Conhecimento: `knowledge_sources`, `documents`, `document_chunks` (embedding `vector(N)` + índice IVFFlat + `tsvector` gerado GIN), `products`, `services`, `faqs`, `business_rules`, `ai_personalities`.
Operação: `leads`, `appointments`, `handoffs`.
Billing/plataforma: `plans`, `subscriptions`, `usage_records`, `llm_usage`.

Convenções: UUID PK (`gen_random_uuid`), `created_at/updated_at` timestamptz, soft delete (`deleted_at`) onde faz sentido, índices por `organization_id`, FKs com `ON DELETE CASCADE/SET NULL` conforme semântica. Migração única inicial: `alembic/versions/0001_initial.py`.

## 5. Decisões técnicas

| Tema | Decisão | Justificativa |
|---|---|---|
| Backend | FastAPI + Pydantic v2 + SQLAlchemy 2 async | async nativo, tipagem, ecossistema |
| Banco | PostgreSQL 16 + pgvector + pg_trgm | RAG relacional sem serviço extra |
| Fila | ARQ (Redis) | async, leve; abstraída por TaskQueue para trocar |
| Auth | JWT curto + refresh rotativo persistido (hash) + revogação por família | logout/roubo de token tratados |
| Senhas | bcrypt (12 rounds) | padrão consolidado |
| LLM | Interface `LLMProvider`; backends: `echo` (dev), `rule`, `openai-compat`, `ollama` | sem vendor lock-in; CI sem chave |
| Embeddings | Interface `EmbeddingProvider`; `hash` (determinístico p/ dev/CI), `openai`, `ollama` | mesmo contrato |
| WhatsApp | Interface `WhatsAppProvider`; adapter Meta Cloud API + fake p/ dev/CI | documentado, trocável |
| Frontend | Next.js (App Router) + TypeScript + Tailwind | padrão de mercado, mantenível |
| Proxy/TLS | Traefik v3 (ACME) em prod compose | HTTPS automático em VPS |
| CI | GitHub Actions: ruff → mypy → unit → integration (serviços) → build | deploy manual |

## 6. Segurança (resumo; detalhes em SECURITY.md)

JWT HS256 (segredo ≥32 bytes), refresh rotativo com detecção de reuso (revoga família), permissões RBAC server-side por recurso, rate limiting, headers de segurança, CORS restrito, webhook com HMAC, segredos só por env, auditoria de eventos sensíveis, isolamento por tenant com testes de BOLA/IDOR.

## 7. Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Vazamento entre tenants | Tenant derivado de token+membership; repos filtram por org; RLS (fase 2); testes de isolamento obrigatórios |
| Alucinação de preços/prazos | Perguntas de fato → consulta estruturada/tools; validador de resposta; instrução de abstenção; handoff quando sem informação |
| Prompt injection | Mensagem do usuário tratada como dado não confiável; system prompt protegido; testes automatizados de injection |
| Webhook duplicado/out-of-order | Unique `external_message_id`/`external_event_id`; idempotência no processamento |
| Custo LLM descontrolado | `llm_usage` por org; quotas por plano; debounce; contexto limitado |
| indisponibilidade LLM/Redis/PG | Falhas de pipeline → retry com backoff + DLQ + status `FAILED`; webhook responde rápido e independe da IA |
| LGPD | Retenção configurável, exclusão/exportação por cliente (fase de hardening LGPD), auditoria |

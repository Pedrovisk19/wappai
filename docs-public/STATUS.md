# STATUS — WAppAI (implementação)

Estado do produto no momento da publicação desta vitrine.

## Fases concluídas (1–9) ✅

| # | Fase | Evidência |
|---|---|---|
| 1 | Fundação (FastAPI modular, settings, logging) | `app.main` assemble |
| 2 | Banco + migrações (Postgres 16 + pgvector, RLS baseline) | Alembic |
| 3 | Auth + RBAC + registro multi-tenant + reset de senha | unit/integration |
| 4 | WhatsApp adapter (Cloud API) + webhook assinado + fila abstrata | e2e docker |
| 5 | Memória (janela + resumo + shopper persistente) + LGPD purge | unit/integration |
| 6 | Knowledge + RAG híbrido (pgvector + FTS + RRF) | unit/integration |
| 7 | Agent Engine (LLM JSON + fallback seguro + prompt builder) | unit |
| 8 | Tools org-scoped + policies + anti-injeção §22 | unit + e2e |
| 9 | Handoff humano: fila → claim → resolve → reativação IA | e2e |

## Produção-like (demonstração real)

- Número comercial conectado (WhatsApp Cloud API) com System User token permanente
- Webhook verificado + campo messages assinado; HMAC opcional documentado
- Pipeline E2E: inbound → RAG → LLM → resposta ~1,5s → persistido no painel
- Handoff operando: fila idempotente → inbox

## Próximas fases (roadmap interno)

- **10** Dashboard analítico (métricas de valor: autoatendimento %, handoff rate)
- **11** Playground dedicado (hoje embutido nas settings)
- **12** Analytics completo
- **13** Billing sobre `usage_records`/`llm_usage` (fundação §40 implementada)
- **14** Wizard de onboarding no painel (hoje: playbook + API)
- **15** Multi-tenant webhook (fallback por phone_number_id)

*Este arquivo resume o estado — os detalhes operacionais (runbook, playbooks de
implantação) permanecem no repositório privado.*

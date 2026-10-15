# Camada WhatsApp — como funciona de ponta a ponta

> Este documento explica a integração WhatsApp do projeto (fake + Meta Cloud API),
> o fluxo de mensagens e as operações que NÃO são óbvias (troca de token, rebind de
> conversas, pegadinhas). Complementa `docs/ARCHITECTURE.md`.

## 1. Visão geral

```
Cliente (WhatsApp real)
   │  mensagem chega no número comercial da org
   ▼
Meta Cloud API (Graph v21.0) ──POST──▶  GET/POST /v1/whatsapp/webhook/{org_slug}
                                           │  (público — autenticação por HMAC/verify token)
                                           ▼
                                    rotas em app/api/v1/routers/whatsapp.py
                                    HMAC (x-hub-signature-256) → META_APP_SECRET (opcional em dev)
                                           │  evento idempotente registrado em whatsapp_webhook_events
                                           ▼
                                    _resolve_account_id(org, parsed)  → conta da org
                                    (match por phone_number_id do payload; fallback: conta fake única)
                                           │  enqueue (inline ou ARQ)
                                           ▼
                                    pipeline_service.process_inbound_event
                                        • dedupe por external_message_id
                                        • upsert de customer (por telefone)
                                        • conversa (re)aproveitada/criada — guarda whatsapp_account_id
                                        • debounce (agrega mensagens da org)
                                           ▼
                                    debounce_flush (pipeline_service.py)
                                        • monta contexto + personalidade + tools + RAG
                                        • LLM (JSON contract: response/handoff/…)
                                        • build_provider(account.provider, credenciais decriptadas)
                                        • send_text → registro em messages (ai_generated=true)
                                        • handoff → fila (WaitHuman) quando for o caso
```

## 2. Os dois provedores e onde ficam as credenciais

O provider é decidido **por conta** (`whatsapp_accounts.provider`), não por env:

| provider | status inicial | credenciais |
|---|---|---|
| `fake` | `connected` | nenhuma (dev/CI) — responde co "fake-text-…" |
| `meta_cloud` | `pending` | `{phone_number_id, access_token, waba_id}` — **criptografadas** (Fernet, chave derivada de `DEVICE_SECRET`) |

- O painel (:3000/settings) **só cria `fake`** (prompt de nome/número).
- Contas `meta_cloud` são criadas via API (Swagger `:8000/docs`): `POST /v1/whatsapp/accounts`
  com `provider: "meta_cloud"` e `credentials: {...}`. A resposta traz `verify_token`
  **uma única vez** (hash fica no banco).
- O env `WHATSAPP_PROVIDER_BACKEND` **não é usado** no fluxo de envio (legado).

## 3. Webhook (Meta)

- **Verificação (GET)**: Meta chama com `hub.mode=subscribe` + `hub.verify_token` +
  `hub.challenge`. O token precisa bater com o global `META_VERIFY_TOKEN` (env) OU com
  o hash SHA-256 de `webhook_verify_token_hash` da conta (~`whatsapp_service.verify_hub_challenge`).
- **Eventos (POST)**: assinatura HMAC validada **somente se** `META_APP_SECRET` estiver
  preenchida (vazio = desligado, aceitável apenas local).
- URL a registrar no Meta App Dashboard: `https://<host>/v1/whatsapp/webhook/{org_slug}`
  — `org_slug` é o slug da ORGANIZAÇÃO (ex.: `padaria-demo`). Campos: assinar `messages`.

## 4. Credenciais

Vários tipos de token da Meta, **importante diferenciá-los** (teste rápido: `GET /me`
— token de usuário devolve o nome da pessoa; token de System User devolve o nome do
usuário do sistema):

| Token | Validade | Uso |
|---|---|---|
| "Temporary access token" (API Setup) | ~24h | testes rápidos |
| User token de longa duração (fb_exchange_token) | 60d | paliativo |
| **System User token** (`business.facebook.com/settings`) | pode ser **nunca expira** | **recomendado** |

Gerar System User: criar usuário do sistema (role Admin) → atribuir o **app** como
asset (Admin) → gerar token com expiração "Nunca" + permissões
`whatsapp_business_messaging` e `whatsapp_business_management`.
Falhar o passo "assign asset" gera token inútil (sem acesso de envio).

## 5. Pegadinhas hard-won (não perca tempo de novo)

### 5.1 Conta trocada → envio caiu no fake
- `whatsapp_accounts` tem constraint **UNIQUE (org, phone_number)** — o DELETE é *soft*
  → recriar a MESMA conta exige remover fisicamente a linha soft-deleted (psql).
- Conversas guardam `whatsapp_account_id` no momento da criação. Se a conta morre
  (hard-delete), conversas antigas ficam com ponteiro morto/NULL e o envio cai
  **silenciosamente no FakeWhatsAppProvider** (sinal nos logs: `out_external_id: fake-text-…`).
- **Rebind**: `UPDATE conversations SET whatsapp_account_id='<novo id>'
  WHERE deleted_at IS NULL AND (whatsapp_account_id IS NULL OR <> '<novo id>');`

### 5.2 `pipeline.replied` com `provider: FakeWhatsAppProvider`
Não é erro de rede: é o fallback do `debounce_flush` (`build_provider('fake')`) quando
a conta não é encontrada. Veja `docs/RUNBOOK.md` para o diagnóstico completo.

### 5.3 Janela de 24h e número restrito
- O agente responde **quem escreveu primeiro** (janela de 24h). Iniciar conversa
  (business-initiated) exige template aprovado pela Meta.
- Número de TESTE da Meta: só envia para até 5 recipients aprovados; inbound chega de
  qualquer pessoa. Número real comercial: sem esse limite.
- Qualidade do número (`quality_rating`): UNKNOWN → capacidade limitada; GREEN após uso.

## 6. Limitações conhecidas (roadmap)

- `debounce_flush` usarp fallback `fake` silencioso quando a conta não existe — ideal:
  buscar a última conta VIVA da org (ou falhar alto).
- `POST /v1/whatsapp/accounts` não suporta atualizar credenciais (hoje precisa delete
  + recriar, disparando 5.1). Um `PATCH .../credentials` eliminaria a dança.
- Painel não tem formulário de incorporação de credenciais `meta_cloud`.

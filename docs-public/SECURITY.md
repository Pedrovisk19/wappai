# Segurança

## Controles implementados (FASE 1) e planejados por fase

### Autenticação
- Senhas: bcrypt 12 rounds (`core/security.py`).
- Access token JWT HS256 curto (30 min) com claims `sub`, `org`, `role`, `sadm`.
- Refresh token opaco (48 bytes URL-safe), armazenado **somente como SHA-256** em `refresh_sessions`, **rotacionado a cada uso**.
- Reuso de refresh já rotacionado → revogação de toda a família (detecção de roubo).
- Logout revoga a família inteira; troca/atribuição de perfil revoga sessões.
- Tenant derivado do token **e revalidado** contra membership ativa no banco a cada request.

### Autorização (RBAC)
- Permissões por recurso em código (`domain/entities.ROLE_PERMISSIONS`): VIEWER < AGENT < MANAGER < ADMIN; SUPER_ADMIN bypass.
- Endpoints declaram `require_permission("recurso:acao")`; decisão **sempre** no backend.

### Isolamento multi-tenant (FASE 2)
- **Role restrita de aplicação:** a API/worker conectam como `wappai_app` (non-superuser, sem ownership). Migrações usam `DATABASE_ADMIN_URL` (dono/superuser).
- **Row Level Security com `ENABLE + FORCE`** em todas as tabelas de tenant (`0002_rls`): políticas negam por padrão e só liberam linhas quando
  `organization_id = current_setting('app.current_org')`.
- **GUCs por transação** (`set_config(..., local=true)`): `app.current_org`, `app.current_user_id`, `app.bypass_rls` — definidos pelo auth dependency (`deps.py`) e pelos serviços via porta `TenantSessionPort` (`infrastructure/db/tenant.py`).
- **Organizações:** cada tenant só lê a própria linha de `organizations`; usuários leem suas memberships via `app.current_user_id` (necessário no login).
- **Bypass explícito** apenas em provisionamento (self-register, seed, jobs de plataforma) — trade-off documentado: o RLS protege contra *bugs de aplicação* (queries sem filtro de org); a role de app tecnicamente pode ligar o bypass, então RLS é segunda barreira, não substituta da checagem em `deps.py`/repositórios.
- **Provas automatizadas** (`tests/integration/test_tenant_isolation.py`, markers `integration isolation`): token de A não acessa org B (404), members não vazam, AGENT não gerencia usuários (403), linhas de clientes invisíveis cross-tenant no banco, INSERT cross-tenant bloqueado (42501).

### Webhook WhatsApp
- Assinatura `X-Hub-Signature-256` (HMAC-SHA256 com `META_APP_SECRET`) verificada antes de qualquer processamento.
- Idempotência: `whatsapp_webhook_events.external_event_id` UNIQUE + `messages.external_message_id` UNIQUE.
- Resposta 200 imediata; processamento na fila.

### Headers/CORS
- `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, CSP default-src 'none' (API).
- CORS allow-list por env (`ALLOWED_ORIGINS`), credenciais habilitadas.

### Segredos
- Somente via env (`.env` fora do git; `.env.example` documentado). Dev usa segredos marcados `insecure-dev-*` que **nunca** devem chegar a staging/prod.

### Planejado nas próximas fases
- Fase 2: RLS PostgreSQL (`app.current_org`) como segunda barreira; rate limiting Redis por IP+rota; auditoria completa de mutações.
- Fase 4: criptografia simétrica das credenciais de WhatsApp por organização (`credentials_encrypted`).
- Fase 7: política anti-injection no AgentEngine; testes automatizados de prompt injection.
- Fase 13: bateria completa BOLA/IDOR por recurso.

## Testes de segurança (obrigatórios)
- `tests/unit/test_security.py` — hashing, tokens, rotação.
- `tests/integration/test_tenant_isolation.py` (marker `isolation`) — BOLA entre orgs em todos os recursos.
- `tests/integration/test_auth_api.py` (marker `integration`) — login/refresh/logout/reuso.

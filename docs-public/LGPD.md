# LGPD — privacidade por desenho

## Princípios aplicados
- **Minimização:** coletamos apenas nome, telefone, e-mail opcional, conteúdo de mensagens e metadados de operação.
- **Finalidade:** dados de clientes finais pertencem à organização (controladora); a plataforma atua como operadora. Não usamos dados de tenants para treinar modelos.
- **Segurança:** criptografia em trânsito (TLS), segredos por env, RBAC, auditoria (`audit_logs`).

## Ciclo de vida e retenção (configurável por organização)
| Dado | Retenção padrão | Onde |
|---|---|---|
| Mensagens | 365 dias | `RETENTION_MESSAGE_DAYS` |
| Resumos de conversa | 180 dias | `RETENTION_SUMMARY_DAYS` |
| Memórias de cliente | 180 dias | `memory_retention_days` (org settings) |
| Auditoria | 365 dias | tabela `audit_logs` |
| Documentos RAG | até exclusão pela org | soft delete + purge |

Worker de retenção (fase de hardening) executa purge diário; exclusão de cliente cascata memórias/conversas via FK.

## Direitos do titular (implementação administrativa)
- **Exportação:** endpoint de export por organização (JSON do cliente + conversas + documentos) — fase LGPD.
- **Exclusão:** exclusão lógica imediata + purge físico no ciclo de retenção; relatório de exclusão em `audit_logs`.
- **Acesso/contestação:** canais da organização; plataforma fornece trilha completa em `audit_logs`.

## Encarregado e base legal
- Base legal típica: execução de contrato/serviço e legítimo interesse da organização controladora; consentimento do titular é responsabilidade da organização (ex.: mensagem de boas-vindas/opt-out no WhatsApp).
- Documente o DPO da organização em `organization_settings` (fase LGPD).

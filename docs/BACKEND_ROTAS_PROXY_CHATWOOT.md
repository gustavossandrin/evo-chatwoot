# Rotas do backend (BFF) — proxy e orquestração sobre Chatwoot

Documento para implementação das **rotas da tua API**. O browser chama só o teu backend; o backend chama a **Application API** do Chatwoot (`/api/v1`) com `api_access_token`. Base Chatwoot (Docker Compose deste repo): `CHATWOOT_INTERNAL_BASE_URL` → tipicamente `http://rails:3000`.

Complementa: `docs/PLANO_EXECUCAO_FRONTEND_CHATWOOT.md`.

---

## 1. Convenções

| Item | Sugestão |
|------|-----------|
| Prefixo API | `/api/v1` ou `/v1` (único na tua app) |
| Auth do cliente | JWT / sessão / API key **tua** — nunca enviar token Chatwoot ao front |
| Escopo Chatwoot | Todas as rotas abaixo assumem `account_id` resolvido no servidor (claims, tenant, ou path `/accounts/{account_id}/...`) |
| Header upstream | `api_access_token`, `Accept: application/json`, `Content-Type: application/json` (exceto multipart em mensagens com anexo) |

Upstream base: `{CHATWOOT_INTERNAL_BASE_URL}/api/v1/accounts/{account_id}`.

---

## 2. Perfil e bootstrap

| Rota tua (exemplo) | Método | Upstream Chatwoot | Notas |
|--------------------|--------|-------------------|--------|
| `/me/chatwoot` ou `/profile` | GET | `GET /api/v1/profile` | Validar utilizador; obter contas e permissões Chatwoot |
| `/accounts` | GET | `GET /api/v1/profile` (derivar lista) ou endpoint de contas conforme versão | Lista de `account_id` que o agente pode usar |

---

## 3. Inboxes (escolher canal de entrada)

| Rota tua | Método | Upstream Chatwoot |
|----------|--------|-------------------|
| `/accounts/{account_id}/inboxes` | GET | `GET .../inboxes` |
| `/accounts/{account_id}/inboxes/{inbox_id}` | GET | `GET .../inboxes/{id}` |
| `/accounts/{account_id}/inboxes/{inbox_id}/agents` | GET | Lista de agentes do inbox (ver doc Inboxes → List Agents in Inbox) |

**Resposta ao front:** normalizar `channel_type`, `id`, `name`, `avatar_url`, `phone_number` quando existir.

---

## 4. Conversas

| Rota tua | Método | Upstream Chatwoot |
|----------|--------|-------------------|
| `/accounts/{account_id}/conversations` | GET | `GET .../conversations` + query (`inbox_id`, `status`, `assignee_type`, paginação — confirmar nomes na tua versão) |
| `/accounts/{account_id}/conversations/meta` | GET | `GET .../conversations/meta` |
| `/accounts/{account_id}/conversations/filter` | POST | `POST .../conversations/filter` body com filtros avançados |
| `/accounts/{account_id}/conversations/{conversation_id}` | GET | Detalhe da conversa (endpoint “conversation details” conforme doc) |
| `/accounts/{account_id}/conversations` | POST | `POST .../conversations` — criar nova conversa (`source_id`, `inbox_id`, opcional `contact_id`, `message`, `status`, `assignee_id`, `team_id`) |
| `/accounts/{account_id}/conversations/{conversation_id}` | PATCH | `PATCH .../conversations/{id}` |
| `/accounts/{account_id}/conversations/{conversation_id}/custom_attributes` | PATCH | `PATCH .../custom_attributes` |
| `/accounts/{account_id}/conversations/{conversation_id}/toggle_status` | POST | `POST .../toggle_status` |
| `/accounts/{account_id}/conversations/{conversation_id}/toggle_priority` | POST | `POST .../toggle_priority` |
| `/accounts/{account_id}/conversations/{conversation_id}/assignments` | POST | Atribuir agente/equipa (Assign Conversation) |
| `/accounts/{account_id}/conversations/{conversation_id}/labels` | GET / POST | List / add labels na conversa |
| `/accounts/{account_id}/conversations/{conversation_id}/typing` | POST | Toggle typing (opcional UX) |
| `/accounts/{account_id}/conversations/{conversation_id}/reporting_events` | GET | Métricas SLA (opcional) |

---

## 5. Mensagens (texto, notas, templates WhatsApp, anexos)

| Rota tua | Método | Upstream Chatwoot |
|----------|--------|-------------------|
| `/accounts/{account_id}/conversations/{conversation_id}/messages` | GET | `GET .../conversations/{id}/messages` query `after`, `before` |
| `/accounts/{account_id}/conversations/{conversation_id}/messages` | POST | `POST .../conversations/{id}/messages` JSON: `content`, `message_type`, `private`, `content_type`, `template_params` (WhatsApp), etc. |
| `/accounts/{account_id}/conversations/{conversation_id}/messages` | POST (multipart) | Mesmo path Chatwoot: `multipart/form-data` com `attachments[]`, `content`, `message_type` — validar com Postman/cURL na tua versão |
| `/accounts/{account_id}/conversations/{conversation_id}/messages/{message_id}` | PATCH | Atualizar mensagem |
| `/accounts/{account_id}/conversations/{conversation_id}/messages/{message_id}` | DELETE | Apagar mensagem e anexos |

---

## 6. Contactos

| Rota tua | Método | Upstream Chatwoot |
|----------|--------|-------------------|
| `/accounts/{account_id}/contacts` | GET | `GET .../contacts` (paginação) |
| `/accounts/{account_id}/contacts/search` | GET | `GET .../contacts/search?q=` |
| `/accounts/{account_id}/contacts/filter` | POST | `POST .../contacts/filter` |
| `/accounts/{account_id}/contacts` | POST | `POST .../contacts` |
| `/accounts/{account_id}/contacts/{contact_id}` | GET | `GET .../contacts/{id}` |
| `/accounts/{account_id}/contacts/{contact_id}` | PATCH | `PATCH .../contacts/{id}` |
| `/accounts/{account_id}/contacts/{contact_id}` | DELETE | `DELETE .../contacts/{id}` |
| `/accounts/{account_id}/contacts/merge` | POST | `POST .../contacts/merge` |
| `/accounts/{account_id}/contacts/{contact_id}/conversations` | GET | Conversas do contacto |
| `/accounts/{account_id}/contacts/{contact_id}/labels` | GET / POST | Labels do contacto |
| `/accounts/{account_id}/contacts/{contact_id}/contact_inboxes` | POST | Associar inbox ao contacto |

---

## 7. Equipas, labels globais, atributos personalizados

| Rota tua | Método | Upstream Chatwoot |
|----------|--------|-------------------|
| `/accounts/{account_id}/teams` | GET | `GET .../teams` |
| `/accounts/{account_id}/teams/{team_id}` | GET | Detalhe equipa |
| `/accounts/{account_id}/agents` | GET | Agentes da conta (ver Agents na doc) |
| `/accounts/{account_id}/custom_attributes` | GET | Definição de campos custom |
| `/accounts/{account_id}/custom_filters` | GET / POST / PATCH / DELETE | Filtros guardados do utilizador |

---

## 8. Webhooks de conta (eventos para o teu backend)

| Rota tua | Método | Upstream Chatwoot |
|----------|--------|-------------------|
| `/accounts/{account_id}/webhooks` | GET | Listar |
| `/accounts/{account_id}/webhooks` | POST | Criar subscrição (URL **pública** do teu servidor que recebe eventos Chatwoot) |
| `/accounts/{account_id}/webhooks/{webhook_id}` | PATCH / DELETE | Atualizar / remover |

Útil para Empurrar atualizações ao frontend (WS/SSE) sem ActionCable no browser.

---

## 9. Relatórios e automação (opcional)

| Rota tua | Upstream | Notas |
|----------|----------|--------|
| `/accounts/{account_id}/reports/...` | Endpoints sob Reports na doc | Muitos exigem perfil administrador |
| `/accounts/{account_id}/automation_rules` | Automation rules | Enterprise / flags conforme instalação |

---

## 10. Ordem sugerida de implementação (MVP → completo)

1. **Bootstrap:** profile + lista de `account_id`.
2. **Inboxes:** GET lista → o front escolhe `inbox_id`.
3. **Conversas:** GET lista + meta + GET detalhe + GET mensagens.
4. **Enviar mensagem:** POST texto (`outgoing`).
5. **Contactos:** GET lista + GET/PATCH + search/filter.
6. **Criar conversa:** POST conversations com `source_id` + `inbox_id`.
7. **Operações:** assign, toggle status, labels.
8. **Anexos:** POST multipart.
9. **Webhooks:** receber eventos no teu servidor e notificar clientes.

---

## 11. Referências oficiais Chatwoot

- Índice da documentação: [developers.chatwoot.com/llms.txt](https://developers.chatwoot.com/llms.txt)
- OpenAPI: [developers.chatwoot.com/api-reference/openapi.json](https://developers.chatwoot.com/api-reference/openapi.json)
- Introdução às APIs: [Introduction to Chatwoot APIs](https://developers.chatwoot.com/api-reference/introduction)

Fixar a **versão/imagem** do Chatwoot em produção e validar query params contra o Swagger dessa tag.

---

**Fim do documento.**

# Webhooks do Chatwoot — referência para o backend

Documento para implementação do **endpoint do teu backend** que recebe os webhooks do Chatwoot. Fonte oficial: [chatwoot.com/docs/product/others/webhook-events](https://www.chatwoot.com/docs/product/others/webhook-events) + OpenAPI ([Add a webhook](https://developers.chatwoot.com/api-reference/webhooks/add-a-webhook)).

---

## 1. Como funciona

- Chatwoot envia um **POST** com **JSON** para a URL configurada.
- Configuração: **Settings → Integrations → Webhooks** ou via API (`POST /api/v1/accounts/{account_id}/webhooks`).
- O segredo é exibido **uma vez** ao criar/editar o webhook.
- Cada evento tem o campo **`event`** com o nome do evento + os atributos do recurso correspondente.

URL típica para este Compose: receber em `https://teu-backend/hooks/chatwoot` (ou rota interna `http://teu-backend:porta/hooks/chatwoot` se ambos estão na mesma rede Docker).

---

## 2. Eventos suportados

Lista oficial dos `subscriptions` aceites pelo Chatwoot:

| Evento | Quando dispara |
|--------|----------------|
| `conversation_created` | Nova conversa criada |
| `conversation_updated` | Mudança em qualquer atributo da conversa (envia `changed_attributes`) |
| `conversation_status_changed` | Status muda (open / resolved / pending / snoozed) |
| `message_created` | Nova mensagem (de contacto, agente ou bot) |
| `message_updated` | Mensagem editada / status alterado (ex.: sent → failed) |
| `contact_created` | Contacto criado |
| `contact_updated` | Contacto alterado |
| `webwidget_triggered` | Visitante abre o widget de chat no site |
| `conversation_typing_on` | Agente começa a escrever (inclui notas internas) |
| `conversation_typing_off` | Agente para de escrever |

> **Nota:** se usares **AgentBot APIs** em vez de webhooks, o `conversation_status_changed` **não** é entregue.

---

## 3. Cabeçalhos HTTP recebidos (assinatura)

Cada POST traz cabeçalhos para verificares autenticidade:

| Header | Conteúdo |
|--------|----------|
| `X-Chatwoot-Signature` | `sha256=<hex>` HMAC-SHA256 do payload |
| `X-Chatwoot-Timestamp` | Unix timestamp (segundos) usado no cálculo |
| `X-Chatwoot-Delivery` | ID único da entrega (quando disponível) |

**Fórmula:**

```
sha256 = HMAC-SHA256( webhook_secret , "{timestamp}.{raw_body}" )
```

### Verificação (Python)

```python
import hmac
import hashlib

def verify_signature(raw_body: bytes, timestamp: str, received_signature: str, secret: str) -> bool:
    message = f"{timestamp}.".encode() + raw_body
    expected = "sha256=" + hmac.new(secret.encode(), message, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, received_signature)
```

### Boas práticas obrigatórias

- Usar **raw body** (bytes do POST) — **não** parsear e re-serializar JSON.
- Comparar com **constant-time** (`hmac.compare_digest` ou equivalente).
- Rejeitar requests com `X-Chatwoot-Timestamp` mais antigo que **~5 minutos** (anti-replay).
- Sempre responder **2xx rápido** (idealmente `204 No Content`); processar pesado em fila.
- Tratar **idempotência** com `X-Chatwoot-Delivery` ou `(event, id, updated_at)`.

---

## 4. Objetos comuns no payload

### Account

```json
{ "id": 1, "name": "Acme" }
```

### Inbox

```json
{ "id": 11, "name": "WhatsApp Vendas" }
```

### Contact

```json
{
  "id": 100,
  "name": "Maria",
  "avatar": "https://...",
  "type": "contact",
  "account": { "id": 1, "name": "Acme" }
}
```

### User (agente)

```json
{ "id": 5, "name": "Joao", "email": "joao@acme.com", "type": "user" }
```

### Conversation

```json
{
  "additional_attributes": {
    "browser": {
      "device_name": "string",
      "browser_name": "string",
      "platform_name": "string",
      "browser_version": "string",
      "platform_version": "string"
    },
    "referer": "string",
    "initiated_at": { "timestamp": "ISO-8601" }
  },
  "can_reply": true,
  "channel": "Channel::Api",
  "id": 22,
  "inbox_id": 11,
  "contact_inbox": {
    "id": 135,
    "contact_id": 100,
    "inbox_id": 11,
    "source_id": "8dfb49f1-...",
    "created_at": "2026-04-18T13:00:00Z",
    "updated_at": "2026-04-18T13:05:00Z",
    "hmac_verified": false
  },
  "messages": [ /* Message[] */ ],
  "meta": {
    "sender": { /* Contact */ },
    "assignee": { /* User */ }
  },
  "status": "open",
  "unread_count": 0,
  "agent_last_seen_at": 0,
  "contact_last_seen_at": 0,
  "timestamp": 1776518762,
  "account_id": 1
}
```

### Message

```json
{
  "id": 5543,
  "content": "init",
  "message_type": 1,
  "created_at": 1776518762,
  "private": false,
  "source_id": null,
  "content_type": "text",
  "content_attributes": {},
  "sender": {
    "type": "user",
    "id": 1, "name": "Joao", "email": "joao@acme.com"
  },
  "account":      { /* Account */ },
  "conversation": { /* Conversation */ },
  "inbox":        { /* Inbox */ }
}
```

**`message_type`** (Application API):

| Valor | Significado |
|-------|-------------|
| `0` | incoming (do contacto) |
| `1` | outgoing (do agente / bot) |
| `2` | activity (sistema) |
| `3` | template (ex.: WhatsApp template) |

> O guia de webhooks documenta o campo como **string** (`"incoming" / "outgoing" / "template"`), mas em **`message_created`/`message_updated`** que vêm do core, costuma chegar como **inteiro**. Trata os dois.

**`content_type`**: `text`, `input_select`, `cards`, `form`, `article`, `input_email`, etc.

**`status` da mensagem** (visto em `message_updated`): `sent`, `delivered`, `read`, `failed` — falhas trazem `content_attributes.external_error`.

---

## 5. Exemplo de payload por evento

### `message_created`

```json
{
  "event": "message_created",
  "id": "1",
  "content": "Hi",
  "created_at": "2020-03-03 13:05:57 UTC",
  "message_type": "incoming",
  "content_type": "text",
  "content_attributes": {},
  "source_id": "",
  "sender":  { "id": "1", "name": "Agent", "email": "agent@x.com" },
  "contact": { "id": "1", "name": "contact-name" },
  "conversation": {
    "display_id": "1",
    "additional_attributes": {
      "browser": { "device_name": "Mac", "browser_name": "Chrome",
                   "platform_name": "Macintosh", "browser_version": "80.0",
                   "platform_version": "10.15.2" },
      "referer": "https://example.com",
      "initiated_at": "Tue Mar 03 2020 18:37:38 GMT-0700"
    }
  },
  "account": { "id": "1", "name": "Chatwoot" }
}
```

### `message_updated`

Mesma estrutura de `message_created`. Útil para acompanhar `status` (`sent → failed`, `sent → delivered → read`) e `content_attributes.external_error` quando a entrega ao canal externo falha.

### `conversation_created`

```json
{
  "event": "conversation_created"
  /* + atributos de Conversation (ver §4) */
}
```

### `conversation_updated`

```json
{
  "event": "conversation_updated",
  "changed_attributes": [
    {
      "status": { "previous_value": "open", "current_value": "resolved" }
    }
  ]
  /* + atributos de Conversation */
}
```

### `conversation_status_changed`

```json
{
  "event": "conversation_status_changed"
  /* + atributos de Conversation */
}
```

### `webwidget_triggered`

```json
{
  "event": "webwidget_triggered",
  "id": "",
  "contact": { /* Contact */ },
  "inbox":   { /* Inbox */ },
  "account": { /* Account */ },
  "current_conversation": { /* Conversation */ },
  "source_id": "string",
  "event_info": {
    "initiated_at": { "timestamp": "ISO-8601" },
    "referer": "https://example.com",
    "widget_language": "pt-BR",
    "browser_language": "pt-BR",
    "browser": {
      "browser_name": "Chrome", "browser_version": "120",
      "device_name": "Mac", "platform_name": "Macintosh",
      "platform_version": "14"
    }
  }
}
```

### `conversation_typing_on` / `conversation_typing_off`

```json
{
  "event": "conversation_typing_on",
  "conversation": { /* Conversation */ },
  "user":         { /* User / AgentBot / Captain */ },
  "is_private":   true
}
```

### `contact_created` / `contact_updated`

Payload com os campos do **Contact** (ver §4). `contact_updated` pode trazer `changed_attributes` análogo ao `conversation_updated`.

---

## 6. Esqueleto sugerido para o teu endpoint

Pseudocódigo (linguagem-agnóstico):

```text
POST /hooks/chatwoot
  raw = read_raw_body()
  signature = headers["X-Chatwoot-Signature"]
  timestamp = headers["X-Chatwoot-Timestamp"]
  delivery  = headers.get("X-Chatwoot-Delivery")

  if !signature or !timestamp:        return 400
  if abs(now - timestamp) > 300:      return 400  # anti-replay
  if !verify_signature(raw, ts, sig, SECRET): return 401

  if delivery and seen(delivery):     return 204  # idempotência
  enqueue(raw)                                    # processamento async
  return 204
```

**Worker:**

1. Ler `event` para fazer dispatch.
2. Aplicar o handler correto (criar/atualizar conversa, mensagem, contacto na tua base / cache / WS para o front).
3. Marcar `delivery` como processado.

---

## 7. Subscrição via API (criar webhook por código)

```bash
curl -X POST "{CHATWOOT_BASE}/api/v1/accounts/{account_id}/webhooks" \
  -H "api_access_token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://teu-backend.com/hooks/chatwoot",
    "subscriptions": [
      "conversation_created",
      "conversation_updated",
      "conversation_status_changed",
      "message_created",
      "message_updated",
      "contact_created",
      "contact_updated",
      "webwidget_triggered"
    ]
  }'
```

Resposta inclui `id`, `url`, `subscriptions`, `account_id`. O segredo é exibido após a criação **na UI** (atualmente não vem na resposta JSON em todas as versões — confirmar na tua tag).

---

## 8. Boas práticas finais

- **Logar** sempre o evento + `delivery` + tempo de processamento.
- **Não confiar** apenas no payload do webhook para dados grandes — usar a API para enriquecer (ex.: ir buscar mensagens completas com paginação).
- **Versionar** os handlers (a estrutura pode mudar entre releases do Chatwoot).
- **Fail-safe:** em caso de erro 5xx, Chatwoot **não** garante reentrega persistente — convém ter **reconciliação periódica** via API para inconsistências.

---

## 9. Referências

- [How to use webhooks?](https://www.chatwoot.com/docs/product/others/webhook-events) — eventos, payloads, HMAC
- [Add a webhook (Application API)](https://developers.chatwoot.com/api-reference/webhooks/add-a-webhook)
- [List / Update / Delete webhooks](https://developers.chatwoot.com/api-reference/webhooks/list-all-webhooks)
- [OpenAPI agregado](https://developers.chatwoot.com/api-reference/openapi.json)

**Fim do documento.**

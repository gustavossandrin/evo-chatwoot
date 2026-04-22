# Plano de execução: frontend próprio + backend proxy sobre Chatwoot (Application API)

Documento de referência para implementação por outro agente ou equipe. Objetivo: replicar o fluxo mental do dashboard Chatwoot (inboxes → conversas por inbox → thread → enviar mensagem), com tráfego **Front → Backend próprio → Chatwoot REST → Backend → Front**.

**Versão do doc:** 1.0  
**Premissa:** Chatwoot self-hosted ou cloud com Application API habilitada; usuário agente com Personal Access Token gerado no perfil Chatwoot.

---

## 1. Objetivos e escopo

### 1.1 Objetivo principal

Construir uma SPA (ou equivalente) que permita ao usuário da **sua plataforma**:

1. Ver **todos os inboxes** da conta Chatwoot (vários números/canais antes de ver conversas).
2. Escolher um inbox e ver **lista de conversas** filtrada por esse inbox.
3. Abrir uma conversa e ver **mensagens** + **enviar mensagens** como agente.
4. Opcionalmente: atribuir conversa, labels, status — conforme necessidade de negócio.

### 1.2 Fora de escopo (explicitar ao agente de implementação)

- Expor `api_access_token` do Chatwoot no browser.
- Implementar iframe do painel Chatwoot oficial.
- Substituir Evolution; este plano assume que canais WhatsApp já chegam ao Chatwoot via integração Evolution (ou nativo).

### 1.3 Fonte de verdade

- **Conversas e mensagens:** PostgreSQL do Chatwoot (acessados só via API).
- **Sua plataforma:** autenticação do seu produto, mapeamento usuário ↔ permissões ↔ `account_id` / tokens Chatwoot (ver §6).

---

## 2. Arquitetura lógica

### 2.1 Fluxo de dados obrigatório

```
┌─────────────┐    HTTPS + auth sua     ┌───────────────┐    HTTPS + api_access_token    ┌─────────────┐
│ Seu frontend│ ───────────────────────► │ Seu backend   │ ─────────────────────────────► │  Chatwoot   │
│  (SPA)      │ ◄────────────────────── │ (BFF / API)   │ ◄───────────────────────────── │  Rails API  │
└─────────────┘    JSON próprio          └───────────────┘    JSON Chatwoot                 └─────────────┘
       ▲                                        ▲
       │  Webhook / SSE / WS (opcional)         │
       └────────────────────────────────────────┘
                    eventos normalizados
```

- **Seu frontend** chama apenas **seu backend** (JWT, sessão, API key sua).
- **Seu backend** chama Chatwoot com header `api_access_token`.
- Chatwoot **nunca** é chamado direto pelo browser (CORS + vazamento de token).

### 2.2 Paralelo com o frontend oficial Chatwoot (Vue)

| Chatwoot dashboard (conceito) | Sua aplicação |
|-------------------------------|---------------|
| Seletor de conta / contexto | `account_id` fixo ou escolhido após login |
| Sidebar / lista de inboxes | Tela ou seção **Inboxes** primeiro |
| Lista de conversas filtrada por inbox | Lista com query `inbox_id` |
| Painel de mensagens | View conversa + GET messages + POST message |
| Tempo real | Webhooks → seu backend → WS/SSE para o front (substitui ActionCable do browser→Chatwoot) |

---

## 3. Autenticação Chatwoot (Application API)

### 3.1 Header

Toda chamada da **camada servidor** ao Chatwoot:

```http
api_access_token: <PERSONAL_ACCESS_TOKEN>
Content-Type: application/json
Accept: application/json
```

Token obtido em: **Chatwoot → Perfil → Access Token** (usuário agente/administrador com acesso à conta).

### 3.2 Base URL

```
{CHATWOOT_BASE}/api/v1
```

Exemplo self-hosted: `https://chatwoot.seudominio.com/api/v1`  
Exemplo local Docker: `http://rails:3000/api/v1` (de dentro da rede Docker; do host: `http://localhost:3003/api/v1` se mapeado).

### 3.3 Identificador de conta

Quase todos os recursos são escopados por:

```
/account_id/ → path parameter account_id (inteiro)
```

Obter contas permitidas ao usuário via `GET /profile` (lista `accounts` / `account_id` ativo conforme resposta real da versão).

---

## 4. Catálogo de endpoints (Application API) — ordem de uso na UI

**Nota:** Nomes de query params de paginação/filtro podem variar por versão do Chatwoot. Fixar versão da imagem Chatwoot e cruzar com a [API Reference](https://www.chatwoot.com/api-reference/introduction) ou Postman Collection oficial.

### 4.1 Bootstrap / sessão Chatwoot

| Ordem | Método | Path | Finalidade |
|-------|--------|------|------------|
| B1 | GET | `/profile` | Validar token; obter `id` do usuário Chatwoot; lista de contas/acessos. |

### 4.2 Inboxes (obrigatório antes da lista de conversas — requisito do produto)

| Ordem | Método | Path | Finalidade |
|-------|--------|------|------------|
| I1 | GET | `/accounts/{account_id}/inboxes` | Listar todos os inboxes da conta (cada “número”/canal costuma ser um inbox). |

**Campos úteis na UI (quando presentes no JSON):** `id`, `name`, `channel_type`, `avatar_url`, meta do canal (ex.: telefone em canais WhatsApp). Tratar resposta como array ou objeto envelope conforme API real (`payload` vs raiz).

### 4.3 Conversas

| Ordem | Método | Path | Finalidade |
|-------|--------|------|------------|
| C1 | GET | `/accounts/{account_id}/conversations` | Listar conversas; **filtrar por `inbox_id`** via query string (confirmar nome exato na doc: `inbox_id`, `status`, `assignee_type`, paginação). |
| C2 | GET | `/accounts/{account_id}/conversations/{conversation_id}` | Detalhe da conversa (meta, assignee, labels). |

### 4.4 Mensagens

| Ordem | Método | Path | Finalidade |
|-------|--------|------|------------|
| M1 | GET | `/accounts/{account_id}/conversations/{conversation_id}/messages` | Histórico da thread (paginar se disponível). |
| M2 | POST | `/accounts/{account_id}/conversations/{conversation_id}/messages` | Enviar mensagem do agente. |

**Corpo mínimo texto (ajustar à doc):**

```json
{
  "content": "Texto da resposta",
  "message_type": "outgoing"
}
```

Anexos: seguir fluxo da versão (ex.: direct upload URLs antes do POST se aplicável).

### 4.5 Operações frequentes (opcional por fase)

| Método | Path | Finalidade |
|--------|------|------------|
| POST | `/accounts/{account_id}/conversations/{id}/assignments` | Atribuir a agente |
| POST | `/accounts/{account_id}/conversations/{id}/toggle_status` | Resolver/reabrir (path pode variar) |
| GET | `/accounts/{account_id}/contacts` | Busca de contatos |
| GET | `/accounts/{account_id}/agents` | Lista agentes para atribuição |

---

## 5. Contrato sugerido do seu backend (BFF)

Expõe ao **seu** frontend JSON estável; internamente proxia Chatwoot.

### 5.1 Exemplo de rotas (prefixo `/api/v1` sua aplicação)

| Seu endpoint | Método | Comportamento interno |
|--------------|--------|------------------------|
| `/chat/accounts` ou usar sessão | GET | Resolver `account_id` ativo; opcionalmente espelhar dados do profile. |
| `/chat/inboxes` | GET | `GET Chatwoot .../accounts/{id}/inboxes` |
| `/chat/conversations` | GET | Query: `inbox_id`, `status`, `page` → repassa ao Chatwoot |
| `/chat/conversations/:id` | GET | Detalhe conversa |
| `/chat/conversations/:id/messages` | GET | Lista mensagens |
| `/chat/conversations/:id/messages` | POST | Envia mensagem |

**Regras:**

- Validar que o usuário da sua app tem permissão para aquele `account_id` / inbox.
- Não repassar token Chatwoot ao cliente.
- Normalizar erros Chatwoot (401/403/422/429) para códigos sua API.

### 5.2 Variáveis de ambiente no backend

- `CHATWOOT_BASE_URL`
- Armazenamento seguro de tokens por tenant/usuário (DB cifrado, vault, etc.)

---

## 6. Multi-tenant e múltiplos números

### 6.1 Modelo Chatwoot

- Uma **Account** contém N **Inboxes**.
- Cada inbox WhatsApp (Evolution) = tipicamente **um inbox** com `channel_type` adequado.
- **Vários números** = várias entradas na resposta de `GET .../inboxes`.

### 6.2 Modelo recomendado na sua aplicação

Tabela conceitual (adaptar nomes):

- `tenant_users` — usuários do seu CRM.
- `chatwoot_credentials` — `user_id`, `chatwoot_account_id`, `encrypted_access_token`, `created_at`.
- Opcional: `user_inbox_pins` — favoritos, ordem, labels amigáveis por `inbox_id`.

Regra: **sempre** listar inboxes antes de conversas; `inbox_id` obrigatório na rota de listagem de conversas (UX e performance).

---

## 7. Tempo real

### 7.1 Opção A — MVP

- Polling: `GET messages` a cada N segundos com conversa aberta; lista de conversas com refresh manual ou interval maior.

### 7.2 Opção B — produção

1. Registrar **Webhooks** no Chatwoot (Administração da conta / Integrações conforme UI) apontando para `https://seu-backend.com/hooks/chatwoot`.
2. Eventos úteis: `message_created`, `conversation_updated`, `conversation_status_changed`, `webwidget_triggered` (se usar web).
3. Backend valida assinatura se Chatwoot enviar secret (conforme versão).
4. Backend publica evento para o front via **WebSocket** ou **SSE** (canal por `user_id` ou `conversation_id`).

### 7.3 O que não fazer no MVP

- Conectar o browser ao ActionCable do Chatwoot com token (expõe credenciais e duplica modelo de auth).

---

## 8. Operações e compatibilidade

### 8.1 Versão Chatwoot

- Fixar imagem Docker (`chatwoot/chatwoot:<tag>`) e executar `rails db:migrate` após cada upgrade.
- Migrações pendentes causam 500 em endpoints (ex.: colunas novas em `channel_api`).

### 8.2 Rate limiting

- Chatwoot pode usar Rack Attack; em dev local considerar `ENABLE_RACK_ATTACK=false` ou IPs permitidos (apenas desenvolvimento).

### 8.3 CORS

- Browser não chama Chatwoot diretamente → CORS só entre front e **seu** backend.

---

## 9. Fluxo de telas (wireframe lógico)

1. **Login** (sua app).
2. **Seleção de contexto** (se múltiplas contas Chatwoot por usuário): escolher `account_id`.
3. **Inboxes:** grid/lista com nome, tipo de canal, indicador de não lidas (se agregar no backend).
4. **Conversas:** lista lateral com filtro `inbox_id` selecionado; tabs ou filtros `open` / `resolved`.
5. **Thread:** coluna principal com mensagens + composer.
6. **Estado vazio:** nenhum inbox → mensagem para cadastrar canal no Chatwoot/Evolution.

Deep link sugerido: `/chat?account=1&inbox=5&conversation=123`.

---

## 10. Fases de execução sugeridas (para o agente de IA)

### Fase 0 — Pré-requisitos

- [ ] Chatwoot acessível e migrações aplicadas.
- [ ] Usuário agente com token de API no Chatwoot.
- [ ] Pelo menos um inbox criado (WhatsApp ou API) para testes.

### Fase 1 — MVP “somente leitura”

- [ ] Backend: proxy `GET profile`, `GET inboxes`, `GET conversations` com `inbox_id`, `GET messages`.
- [ ] Frontend: tela inboxes → lista conversas → visualizar mensagens (sem enviar).

**Critério de aceite:** Usuário navega inbox → conversa → vê histórico igual ao Chatwoot para o mesmo `conversation_id`.

### Fase 2 — Envio e refresh

- [ ] `POST message` outgoing.
- [ ] Polling ou primeiro webhook → atualização da lista/timeline.

**Critério de aceite:** Mensagem enviada aparece no Chatwoot oficial e na sua UI.

### Fase 3 — Operações de agente

- [ ] Assign, mudança de status, labels (conforme prioridade).

### Fase 4 — Tempo real completo

- [ ] Webhooks + WS/SSE; remover ou reduzir polling.

---

## 11. Testes manuais (checklist)

- [ ] Token inválido → 401 tratado no seu backend.
- [ ] `account_id` sem permissão → 403.
- [ ] Lista inboxes vazia vs com N inboxes.
- [ ] Troca de inbox limpa lista de conversas e estado da thread.
- [ ] Conversa com muitas mensagens — paginação ou scroll infinito.
- [ ] Envio com anexo (se escopo incluir).

---

## 12. Referências oficiais Chatwoot

- [Introduction to Chatwoot APIs](https://www.chatwoot.com/api-reference/introduction) — Application vs Platform vs Client.
- [Environment variables (self-hosted)](https://www.chatwoot.com/docs/self-hosted/configuration/environment-variables) — `FRONTEND_URL`, `SECRET_KEY_BASE`, etc.
- Postman: workspace “Chatwoot APIs” linkado na documentação.

---

## 13. Notas para o agente executor

1. **Não** embutir token Chatwoot no frontend.
2. Sempre obter **lista de inboxes** antes de assumir `inbox_id` na URL.
3. Confirmar na documentação da **versão exata** do Chatwoot os nomes dos query params de `GET /conversations`.
4. Manter logs de proxy (request id, `account_id`, sem logar token).
5. Se integrar Evolution depois: cada instância/número deve mapear para um **inbox** existente no Chatwoot; este plano não cria inbox pela Evolution, apenas consome os já listados pela API.

---

**Fim do documento.** Ajuste nomes de rotas e fases ao stack da sua empresa (Nest, Laravel, Django, etc.) mantendo o contrato conceitual acima.

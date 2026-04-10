---
name: criar-fluxo
description: Constrói e publica um workflow n8n via API seguindo todos os padrões obrigatórios do projeto. Use esta skill quando o plano estiver aprovado e o usuário disser "pode construir", "cria o fluxo", "bora", "implementar", "subir para o n8n", ou após confirmação explícita no /aprovar-fluxo. Não construa sem plano aprovado — se não houver plano claro, use /planejar-fluxo primeiro.
---

# Skill: Criar Fluxo n8n

Constrói e publica workflows no n8n via API. Só execute após plano aprovado.

## Antes de começar

1. Confirme que o plano foi aprovado (via `/aprovar-fluxo` ou explicitamente pelo usuário)
2. Leia a memória do projeto para carregar credenciais, URL do n8n e padrões
3. Se o workflow já existe no n8n, faça GET para preservar o Webhook ID e o node ID do webhook

## Padrões obrigatórios

### Nomenclatura de nós

| Prefixo | Tipo de nó |
|---|---|
| `TRIGGER –` | Webhooks, schedulers |
| `CODE –` | Code nodes |
| `HTTP –` | HTTP Request (ClickUp, APIs externas) |
| `IF –` | Condicionais |
| `WPP –` | WhatsApp via Zappfy |
| `EMAIL –` | Gmail/SMTP |
| `ERROR –` | Nós de tratamento de erro (quando não for HTTP) |

### CONFIG block (obrigatório no `CODE – Extrair Dados`)

```javascript
// ============================================================
// CONFIGURAÇÕES — edite apenas esta seção ao replicar
// ============================================================
const CONFIG = {
  // ClickUp
  clickupBaseUrl: "https://api.clickup.com/api/v2",
  listId:         "ID_DA_LISTA",
  statusInicial:  "status",
  fieldXxx:       "ID_DO_CAMPO",
  // Zappfy
  zappfyBaseUrl:  "https://YOUR_ZAPPFY_INSTANCE.uazapi.com",
};
// ============================================================
```

**Nunca** coloque tokens, API keys ou IDs de credencial no CONFIG. Esses valores ficam nas credenciais do n8n.

### Referências nos nós HTTP

Todos os valores devem referenciar o CONFIG:
```
={{ $('CODE – Extrair Dados').first().json.clickupBaseUrl + '/list/' + $('CODE – Extrair Dados').first().json.listId + '/task' }}
```

### Credenciais nos nós HTTP (ClickUp)

```json
"authentication": "predefinedCredentialType",
"nodeCredentialType": "clickUpApi",
"credentials": {
  "clickUpApi": {"id": "YOUR_CREDENTIAL_ID", "name": "YOUR_CREDENTIAL_NAME"}
}
```

### Tratamento de erro

**Nós críticos** (Criar Task, Adicionar Comentário):
```json
"onError": "continueErrorOutput"
```
→ Conecte a saída de erro a um nó de fallback ou comentário de erro.

**Nós não críticos** (WPP, EMAIL):
```json
"continueOnFail": true
```
→ Após ambos, adicione `CODE – Checar Erros de Envio` que verifica `$json.error` e posta comentário na task.

### Fallback para criação de task

Sempre que houver `HTTP – Criar Task` (com campos completos), adicione o fallback:
```
HTTP – Criar Task [onError: continueErrorOutput]
  ├→ [ok]   → WPP/EMAIL
  └→ [erro] → HTTP – Criar Task Simples (só nome + status)
                   ├→ [ok]   → HTTP – Comentar Cadastro Incompleto → WPP/EMAIL
                   └→ [erro] → fim (log no n8n)
```

## Processo de construção

### 1. Gerar o JSON do workflow

Use Python para montar o JSON (evita problemas de escaping):
- Defina todos os nós com suas posições, IDs e parâmetros
- Monte o objeto `connections` separadamente
- Salve em `/tmp/workflow_[nome].json`

### 2. Publicar via API

```bash
# Workflow novo:
curl -s -X POST "{n8n_url}/api/v1/workflows" \
  -H "X-N8N-API-KEY: {key}" \
  -H "Content-Type: application/json" \
  -d @/tmp/workflow_[nome].json

# Workflow existente (PUT):
curl -s -X PUT "{n8n_url}/api/v1/workflows/{id}" \
  -H "X-N8N-API-KEY: {key}" \
  -H "Content-Type: application/json" \
  -d @/tmp/workflow_[nome].json
```

O body do PUT aceita apenas: `name`, `nodes`, `connections`, `settings: {executionOrder: "v1"}`, `staticData`.

### 3. Ativar o workflow (se novo)

```bash
curl -s -X POST "{n8n_url}/api/v1/workflows/{id}/activate" \
  -H "X-N8N-API-KEY: {key}"
```

### 4. Verificar

Confirme que os nós foram criados com os nomes corretos:
```bash
curl -s -H "X-N8N-API-KEY: {key}" "{n8n_url}/api/v1/workflows/{id}" | jq '[.nodes[] | {name, type}]'
```

## Após construir

Informe ao usuário:
- Nome e ID do workflow
- URL para ver no n8n
- Instrução para teste: submeter um evento de teste e verificar a execução

Sugira os próximos passos: `/gerar-json` e `/gerar-doc`.

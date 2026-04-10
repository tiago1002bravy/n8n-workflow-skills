---
name: testar-fluxo
description: Executa testes e2e automatizados de um workflow n8n publicado. Roda o workflow, inspeciona cada node, valida a entrega final e repete até passar. Use esta skill quando o workflow foi construido via /criar-fluxo e o usuario pedir "testa", "roda o teste", "valida", "testar-fluxo", "garante que funciona", ou automaticamente apos /criar-fluxo se o usuario pedir teste e2e. Minima intervencao humana — so pergunte ao usuario se estiver bloqueado.
---

# Skill: Testar Fluxo n8n

Executa testes end-to-end em um workflow n8n publicado. Valida node a node e a entrega final. Repete até que tudo passe. Mínima intervenção humana.

## Antes de começar

1. Leia a memória do projeto para carregar URL do n8n, API key e credenciais
2. Identifique o workflow a testar (ID vem da conversa ou do `/criar-fluxo`)
3. Busque o workflow via API para entender a estrutura de nós

```bash
curl -s -H "X-N8N-API-KEY: {key}" "{n8n_url}/api/v1/workflows/{id}" | jq '{name, active, nodes: [.nodes[] | {name, type}]}'
```

## Fase 1 — Validação estrutural (pré-execução)

Antes de executar, valide a estrutura do workflow:

### 1.1 Nomenclatura dos nós

Verifique que todos os nós seguem o padrão:

| Prefixo esperado | Tipos de nó |
|---|---|
| `TRIGGER –` | `n8n-nodes-base.webhook`, `n8n-nodes-base.scheduleTrigger`, `n8n-nodes-base.manualTrigger` |
| `CODE –` | `n8n-nodes-base.code` |
| `HTTP –` | `n8n-nodes-base.httpRequest` |
| `IF –` | `n8n-nodes-base.if` |
| `WPP –` | Nós que chamam Zappfy |
| `EMAIL –` | `n8n-nodes-base.gmail`, `n8n-nodes-base.emailSend` |
| `ERROR –` | Nós de tratamento de erro |

**Critério:** PASS se todos os nós seguem o prefixo correto. WARN se algum nó genérico (ex: "Code1") existir.

### 1.2 CONFIG block

Inspecione o código do nó `CODE – Extrair Dados`:

```bash
curl -s -H "X-N8N-API-KEY: {key}" "{n8n_url}/api/v1/workflows/{id}" \
  | jq '.nodes[] | select(.name | startswith("CODE – Extrair")) | .parameters.jsCode'
```

Verifique:
- Existe um bloco `const CONFIG = { ... }` no início
- Nenhum token, API key ou ID de credencial no CONFIG
- Todos os IDs de listas, campos, URLs base estão no CONFIG

**Critério:** PASS se CONFIG existe e não contém secrets. FAIL se hardcoded.

### 1.3 Credenciais nos nós HTTP

Para cada nó HTTP, verifique:

```bash
curl -s -H "X-N8N-API-KEY: {key}" "{n8n_url}/api/v1/workflows/{id}" \
  | jq '[.nodes[] | select(.type == "n8n-nodes-base.httpRequest") | {name, auth: .parameters.authentication, credType: .parameters.nodeCredentialType, creds: .credentials}]'
```

- `authentication` deve ser `"predefinedCredentialType"` (nunca `"genericCredentialType"` com header hardcoded)
- `credentials` deve ter ID e nome válidos

**Critério:** PASS se todas as credenciais estão configuradas via predefined. FAIL se algum nó usa auth inline.

### 1.4 Conexões

Verifique que todos os nós estão conectados (nenhum nó órfão):

```bash
curl -s -H "X-N8N-API-KEY: {key}" "{n8n_url}/api/v1/workflows/{id}" \
  | jq '{nodes: [.nodes[].name], connected: [.connections | to_entries[] | .key]}'
```

Compare: todo nó (exceto o trigger) deve aparecer como destino em alguma conexão. Todo nó (exceto o último) deve aparecer como origem.

**Critério:** PASS se grafo está conectado. FAIL se há nós órfãos.

### 1.5 Rotas de erro

Para nós marcados como críticos, verifique `onError: "continueErrorOutput"`. Para não críticos, `continueOnFail: true`.

**Critério:** PASS se rotas estão configuradas. WARN se ausentes (depende do tipo de workflow).

### Resultado da Fase 1

Apresente uma tabela resumo:

| Check | Status | Detalhe |
|---|---|---|
| Nomenclatura | PASS/WARN/FAIL | ... |
| CONFIG block | PASS/FAIL | ... |
| Credenciais | PASS/FAIL | ... |
| Conexões | PASS/FAIL | ... |
| Rotas de erro | PASS/WARN | ... |

Se houver FAIL, corrija via API (PUT no workflow) e revalide antes de prosseguir.

## Fase 2 — Execução do workflow

### 2.1 Disparar execução

Dependendo do tipo de trigger:

**Schedule/Manual trigger** — Use a API interna do n8n:
```bash
# Obter cookie de sessão (precisa estar logado no browser)
# Se não tiver cookie, peça ao usuário para fornecer ou usar o n8n manualmente

# Via API interna (requer sessão):
curl -s -X POST "{n8n_url}/rest/workflows/{id}/run" \
  -H "Cookie: {session_cookie}" \
  -H "Content-Type: application/json" \
  -d '{"workflowData": WORKFLOW_JSON_COMPLETO}'
```

**Webhook trigger** — Envie o payload de teste:
```bash
curl -s -X POST "{n8n_url}/webhook-test/{path}" \
  -H "Content-Type: application/json" \
  -d '{PAYLOAD_DE_TESTE}'
```

Se não conseguir disparar via API (sem cookie, sem acesso):
1. Tente buscar a execução mais recente para analisar
2. Se não houver execução, peça ao usuário para executar manualmente UMA VEZ e depois continue a análise

### 2.2 Aguardar e buscar execução

```bash
# Listar execuções recentes do workflow
curl -s -H "X-N8N-API-KEY: {key}" \
  "{n8n_url}/api/v1/executions?workflowId={id}&limit=5&status=success,error,waiting"
```

Pegue a execução mais recente e inspecione:

```bash
curl -s -H "X-N8N-API-KEY: {key}" \
  "{n8n_url}/api/v1/executions/{execution_id}"
```

### 2.3 Analisar resultado node a node

Para cada nó na execução, verifique:

| O que verificar | Como |
|---|---|
| Status do nó | `resultData.runData[nodeName][0].executionStatus` deve ser `"success"` |
| Dados de saída | `resultData.runData[nodeName][0].data.main[0]` deve ter items |
| Erros | Se `executionStatus` é `"error"`, leia `error.message` |
| HTTP status | Para nós HTTP, verifique se o response code é 2xx |

**Para cada nó com erro:**
1. Leia a mensagem de erro completa
2. Identifique a causa raiz (URL errada? campo inválido? credencial expirada? body mal formatado?)
3. Corrija no workflow via PUT na API
4. Volte à Fase 2.1 (re-executar)

### Loop de correção automática

```
ENQUANTO houver nós com erro:
  1. Identifique o primeiro nó que falhou
  2. Diagnostique (leia erro, compare com input esperado)
  3. Corrija (PUT no workflow com nó atualizado)
  4. Re-execute
  5. Re-analise
  
  LIMITE: máximo 5 iterações
  Se atingir o limite, apresente os erros restantes ao usuário
```

## Fase 3 — Validação da entrega final

Após execução bem-sucedida, valide o resultado real no sistema destino.

### 3.1 Identificar o que foi criado

Com base no tipo de workflow, verifique:

| Tipo de entrega | Como validar |
|---|---|
| Task no ClickUp | GET na task criada, verificar campos preenchidos |
| Doc/Page no ClickUp | GET no doc/page, verificar conteúdo e formatação |
| Mensagem WhatsApp | Verificar nos logs do Zappfy ou na execução |
| E-mail enviado | Verificar nos logs do Gmail ou na execução |
| Dados atualizados | GET no recurso atualizado |

### 3.2 Checklist de validação do conteúdo

Para cada entrega, verifique:

- [ ] O recurso foi criado/atualizado no destino correto (lista, space, doc certo)
- [ ] Todos os campos obrigatórios estão preenchidos
- [ ] Formatação está correta (markdown, HTML, texto limpo)
- [ ] Links funcionam (se aplicável)
- [ ] Dados não estão duplicados
- [ ] Nenhum campo com valor "undefined", "null", "[object Object]" ou vazio indevido

### 3.3 Validação de formatação (se entrega é documento/texto)

- Headings: verificar hierarquia correta (não usar H1/H2 se restrito a H3+)
- Tabelas: verificar alinhamento, colunas completas
- Caracteres especiais: acentos, emojis (se proibidos), markdown escapado
- Separadores: `---` pode virar `* * *` em alguns sistemas — verificar renderização

### 3.4 Limpeza pós-teste

Após validar, limpe os artefatos de teste:
- Se criou task de teste → deletar via API (`DELETE /api/v2/task/{id}`)
- Se criou doc/page de teste → tentar deletar via API (ClickUp v3 pode retornar 405 — nesse caso, informar o usuário)
- Se enviou mensagem de teste → apenas registrar (não é reversível)

Tente sempre limpar. Se a API não suportar exclusão, diga ao usuário exatamente o que precisa ser deletado manualmente (nome, link, ID).

## Fase 4 — Relatório final

Apresente o resultado completo:

---

### Resultado do teste: `[Nome do Workflow]`

**Workflow:** [nome] (`[ID]`)
**URL:** [link para o n8n]
**Execuções realizadas:** [N]

#### Validação estrutural

| Check | Status |
|---|---|
| Nomenclatura | PASS/WARN |
| CONFIG block | PASS |
| Credenciais | PASS |
| Conexões | PASS |
| Rotas de erro | PASS/WARN |

#### Execução node a node

| # | Nó | Status | Items | Obs |
|---|---|---|---|---|
| 1 | `TRIGGER – ...` | OK | 1 | — |
| 2 | `CODE – Extrair Dados` | OK | N | — |
| ... | ... | ... | ... | ... |

#### Entrega final

| Item | Status | Detalhe |
|---|---|---|
| [recurso criado] | OK/FALHA | [o que foi verificado] |
| Formatação | OK/FALHA | [detalhes] |
| Campos preenchidos | OK/FALHA | [detalhes] |

#### Correções aplicadas durante o teste
- [lista de correções, se houver — ex: "Corrigido body do HTTP – Criar Page: trocado specifyBody de string para json"]

#### Limpeza
- [o que foi deletado automaticamente]
- [o que precisa ser deletado manualmente, se algo]

---

**Veredicto:** APROVADO / REPROVADO

Se APROVADO: "Workflow testado e funcionando. Próximos passos: `/gerar-json` e `/gerar-doc`."
Se REPROVADO: Liste os problemas pendentes e o que o usuário precisa resolver.

## Notas importantes

- **Mínima intervenção humana:** Só pergunte ao usuário se estiver genuinamente bloqueado (ex: credencial expirada, acesso negado, precisa de cookie de sessão)
- **Limite de iterações:** Máximo 5 ciclos de correção automática. Após isso, escale para o usuário
- **Não altere a lógica do workflow:** Corrija apenas erros técnicos (URL, body, credencial). Se a lógica estiver errada (ex: mapeamento de campo incorreto), reporte ao usuário
- **Preserve o workflow original:** Antes de qualquer correção, anote o estado anterior para poder reverter se necessário
- **Artefatos de teste:** Sempre tente limpar. Se não conseguir, documente no relatório

---
name: planejar-fluxo
description: Guia o planejamento completo de um novo workflow n8n antes de construir qualquer coisa. Use esta skill sempre que o usuário quiser criar uma nova automação, mencionar "novo fluxo", "quero automatizar", "preciso de uma automação que...", "criar workflow", "planejar fluxo", ou descrever uma lógica de integração entre sistemas (ClickUp, WhatsApp, formulários, e-mail, etc.). Não pule para a construção — o objetivo desta skill é mapear o ambiente, fazer as perguntas certas e gerar um plano estruturado para aprovação.
---

# Skill: Planejar Fluxo n8n

Guia o processo de planejamento de um novo workflow antes de qualquer construção.

## Contexto do projeto

Antes de começar, leia a memória do projeto para carregar:
- URL do n8n, credenciais existentes, padrões obrigatórios
- Arquivo: memória do projeto (configure o path na sua instalação)

## Passo 1 — Mapear o ambiente

Faça essas chamadas em paralelo para entender o estado atual:

```bash
# Listar workflows existentes
curl -s -H "X-N8N-API-KEY: {key}" "{n8n_url}/api/v1/workflows?limit=50"

# Listar credenciais disponíveis
curl -s -H "X-N8N-API-KEY: {key}" "{n8n_url}/api/v1/credentials"
```

Apresente ao usuário:
- Workflows já existentes (para evitar duplicidade)
- Credenciais disponíveis (ClickUp, Zappfy, Gmail, etc.)

## Passo 2 — Entender o fluxo

Faça as perguntas necessárias. Nem todas precisam ser respondidas de uma vez — adapte ao contexto. O objetivo é não construir nada errado.

**Gatilho:**
- O que dispara o fluxo? (webhook de formulário, webhook do ClickUp, schedule, manual?)
- Tem payload de exemplo? Se sim, peça para compartilhar ou inspecionar uma execução existente

**Lógica principal:**
- Qual a sequência de ações? (buscar, criar, atualizar, notificar...)
- Existe alguma condição de decisão? (IF: lead existe? tarefa tem campo X?)
- Quais sistemas estão envolvidos? (ClickUp, Zappfy, Gmail, outro)

**ClickUp — se aplicável:**
- Em qual space/folder/lista as tasks serão criadas ou buscadas?
- Quais campos customizados precisam ser preenchidos? (busque via `GET /list/{list_id}/field` para listar IDs e tipos)
- Busca por campos custom? Por quê? (e-mail, nome, ID?)
- Quem será o responsável (assignee) pela task? (obrigatório definir antes de construir)
- Qual o prazo (due_date)? E deve incluir start_date? (sempre usar `_time: true` para preservar hora)

**Notificações — se aplicável:**
- WhatsApp vai para quem? Lead, equipe interna, ambos?
- E-mail vai para quem? Qual conta de envio?
- Qual o texto das mensagens? (pode sugerir modelo)

**Tratamento de erro:**
- Se uma ação crítica falhar, o que deve acontecer?
- Onde registrar erros? (comentário na task, notificação, log)

## Passo 3 — Mapear campos

Se o fluxo envolve ClickUp com campos customizados, busque-os:

```bash
curl -s -H "Authorization: {clickup_token}" \
  "https://api.clickup.com/api/v2/list/{list_id}/field"
```

Monte uma tabela clara:

| Campo do payload | Campo no ClickUp | Tipo | Mapeamento necessário |
|---|---|---|---|
| `answers["Nome"]` | Título da task | texto | — |
| `answers["E-mail"]` | `05. E-mail` | email | — |
| `answers["Situação"]` | `Situação do imóvel` | dropdown | ver opções abaixo |

Para dropdowns, liste as opções e o `orderindex` necessário para a API.

## Passo 4 — Gerar o plano estruturado

Ao ter informações suficientes, gere um plano no formato abaixo. Este plano será entregue ao usuário para aprovação antes de qualquer construção.

---

### Plano: `[area]gatilho->resultado`

> Nome do workflow deve seguir o padrão `[area]gatilho->resultado` — ex: `[comercial]form-preenchido->cadastro-lead`. Área em minúsculo sem espaços, gatilho e resultado em kebab-case.

**Gatilho:** [descrição]
**Lista ClickUp destino:** [nome] (ID: `xxx`)

**Fluxo de nós:**
```
TRIGGER – [nome]
  └→ CODE – Extrair Dados
       └→ [próximos nós...]
```

**Campos mapeados:**
| Dado | Destino no ClickUp | Observação |
|---|---|---|

**Rotas de erro:**
- [nó crítico] → falha → [ação]
- WPP/Email → falha → continua, registra comentário na task

**Credenciais a usar:**
- ClickUp – Produção
- [outras...]

**Perguntas ainda em aberto:**
- [lista o que ainda precisa ser respondido]

---

Ao finalizar o plano, diga: "Plano pronto. Revise e use `/aprovar-fluxo` para aprovar e iniciar a construção."

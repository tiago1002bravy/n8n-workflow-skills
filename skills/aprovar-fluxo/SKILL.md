---
name: aprovar-fluxo
description: Apresenta o plano atual de um workflow n8n de forma estruturada e aguarda aprovação explícita antes de construir. Use esta skill quando o usuário disser "aprovado", "pode construir", "bora", "sim", "vamos", ou quando quiser revisar o plano antes de confirmar. Também use quando terminar o planejamento e for hora de pedir confirmação. Nunca inicie a construção sem passar por esta skill — ela é o checkpoint obrigatório entre planejar e construir.
---

# Skill: Aprovar Fluxo

Checkpoint obrigatório entre o planejamento e a construção. Apresenta o plano completo e aguarda aprovação explícita.

## O que fazer

1. Recupere o plano da conversa atual (gerado pelo `/planejar-fluxo`)
2. Apresente-o no formato estruturado abaixo
3. Aguarde o usuário confirmar explicitamente

Se o plano ainda tiver itens em aberto (perguntas não respondidas), resolva-os antes de pedir aprovação.

## Formato de apresentação

---

## Plano para aprovação: [Nome do Workflow]

### Resumo
- **Gatilho:** [descrição do trigger]
- **Objetivo:** [o que o fluxo faz em 1 frase]
- **Workflow n8n:** [nome] (`[ID se existente]`)

### Nós que serão criados

| # | Nome do nó | Tipo | O que faz |
|---|---|---|---|
| 1 | `TRIGGER – [nome]` | Webhook | Recebe payload do [sistema] |
| 2 | `CODE – Extrair Dados` | Code | CONFIG + extrai e mapeia campos |
| ... | ... | ... | ... |

### Campos mapeados no ClickUp
| Dado recebido | Campo ClickUp | Tipo |
|---|---|---|
| ... | ... | ... |

### Rotas de erro
| Nó | Se falhar | Ação |
|---|---|---|
| `HTTP – Criar Task` | fallback | Cria task simples + comenta campos pendentes |
| `WPP – Enviar Confirmação` | não crítico | Continue on fail + comenta erro na task |
| `EMAIL – Enviar Confirmação` | não crítico | Continue on fail + comenta erro na task |

### Credenciais
- [lista todas as credenciais que serão usadas]

### Padrões aplicados
- ✅ CONFIG block centralizado em `CODE – Extrair Dados`
- ✅ Nenhum valor hardcoded nos nós HTTP
- ✅ Nomenclatura: TRIGGER / CODE / HTTP / IF / WPP / EMAIL / ERROR
- ✅ Credenciais centralizadas
- ✅ Rotas de erro estruturadas

---

**Aguardando aprovação.** Responda "sim" para iniciar a construção, ou faça ajustes no plano antes.

## Após aprovação

Quando o usuário confirmar, diga:
> "Plano aprovado. Use `/criar-fluxo` para iniciar a construção."

Não construa nada nesta skill — ela apenas aprova.

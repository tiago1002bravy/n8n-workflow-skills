# n8n Workflow Skills for Claude Code

4 skills for Claude Code that automate the full lifecycle of n8n workflow creation: **plan > approve > build > test**.

## Skills

| Skill | Command | What it does |
|---|---|---|
| **Planejar Fluxo** | `/planejar-fluxo` | Maps the n8n environment (existing workflows, credentials), asks the right questions, maps fields, and generates a structured plan |
| **Aprovar Fluxo** | `/aprovar-fluxo` | Mandatory checkpoint — presents the plan in a structured format and waits for explicit approval before building |
| **Criar Fluxo** | `/criar-fluxo` | Builds and publishes the workflow to n8n via API following all mandatory patterns (CONFIG block, naming, credentials, error routes) |
| **Testar Fluxo** | `/testar-fluxo` | Runs e2e tests: structural validation, node-by-node execution analysis, final delivery validation, auto-fix loop (max 5 iterations), cleanup |

## Flow

```
/planejar-fluxo  -->  /aprovar-fluxo  -->  /criar-fluxo  -->  /testar-fluxo
```

## Patterns enforced

- **CONFIG block**: All variables isolated in a `CODE - Extrair Dados` node — no hardcoded values in HTTP nodes
- **Naming convention**: `TRIGGER -`, `CODE -`, `HTTP -`, `IF -`, `WPP -`, `EMAIL -`, `ERROR -`
- **Credentials**: Always via `predefinedCredentialType` — never inline tokens
- **Error handling**: Critical nodes use `continueErrorOutput` with fallback; non-critical use `continueOnFail`
- **Fallback**: Task creation always has a simplified fallback path

## Installation

Copy the skill folders into your Claude Code skills directory:

```bash
cp -r skills/* ~/.claude/skills/
```

Then register them in your Claude Code settings (userSettings or project settings) so they appear as slash commands.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI or IDE extension
- n8n instance with API access (API key)
- `curl` and `jq` available in your shell

## Customization

Each skill uses `{n8n_url}` and `{key}` placeholders. Store your actual values in Claude Code project memory so the skills can load them at runtime.

Credential IDs in `criar-fluxo` use `YOUR_CREDENTIAL_ID` / `YOUR_CREDENTIAL_NAME` placeholders — replace with your actual n8n credential IDs.

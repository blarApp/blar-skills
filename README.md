# Blar for Claude

This plugin connects Claude to your company's Blar account and adds the `blar-activation` skill. With
it, Claude can read how your customer accounts activate, what Blar's nudges and AI agent achieve,
and what your users ask. It first agrees with you on what counts as an activated account, measures
your activation on that definition, suggests what to change, and with your confirmation corrects
your Blar documentation and curates your feature catalog.

You need a Blar account. Everything the connector reads or changes belongs only to your company.

## Install

**Claude (web and desktop, paid plans)**

1. Download `blar-plugin.zip` from the [latest release](https://github.com/blarApp/blar-skills/releases/latest).
2. Go to Customize > Plugins > Add > Upload plugin and choose the zip.
3. Open the plugin's Connectors tab, connect Blar, and sign in with your Blar account.
4. On Team and Enterprise plans, an owner can publish the plugin to the organization, as available,
   installed by default or required. Each member then connects Blar with their own account.

To update, upload the zip from the newest release the same way.

**Claude Code**

```
/plugin install blar --marketplace blarApp/blar-skills
```

The plugin brings the connector and the skill. Run `/mcp` and sign in to the Blar connector.

**Other agents (Cursor, Codex, GitHub Copilot and more)**

```bash
npx skills add blarApp/blar-skills
```

This installs only the skill. Add the Blar MCP server to your agent as a remote HTTP server at
`https://api.blar.io/api/mcp` and sign in with your Blar account.

## Permissions

Reading is granted by default. Editing documentation, and reading or editing features, are separate
permissions, approved on Blar's consent screen when you connect. Features also require a Blar
admin account.

## Try

- "How is our activation going?"
- "Show the activation curve by weekly cohort, by segment."
- "How many accounts created 100 products within their first 30 days?"
- "Why did the AI agent hand conversations to the team this month, and what should we fix?"

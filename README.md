# NuaSense skills

Agent skills and a Claude Code plugin for building on [NuaSense](https://api.nuasense.com/index.md): East African farm and market data, live stations and soil probes with the agronomy computed from them, and county prices, satellite demand, news signals and food security for Kenya, Tanzania, Uganda and Rwanda.

## Install the skill

For Claude Code, Codex, Cursor and any other agent that reads `SKILL.md` files, from inside your project:

```bash
npx skills add nuasense/skills
```

The skill teaches the agent how to get a key (the one step that stays with you: the console hands out a key only after a verified phone), how to read the catalog for that key, how to call datasets and analyses, what a unit is, what each error means, how to connect a station of any brand, and how to use the MCP server instead of REST.

## Install the Claude Code plugin

The same skill plus the hosted MCP server, in one plugin:

```text
/plugin marketplace add nuasense/skills
/plugin install nuasense@nuasense
```

The MCP server at `https://mcp.nuasense.com/mcp` signs in with your NuaSense console account; there is no key to paste.

## Install the Gemini CLI extension

The skill and the hosted MCP server for Gemini CLI:

```bash
gemini extensions install https://github.com/nuasense/skills
```

Gemini CLI opens the sign-in page of your NuaSense account on first use.

## Docker MCP Toolkit, VS Code, Cursor

The MCP server is a plain Streamable HTTP endpoint with OAuth sign-in, so any MCP client can add `https://mcp.nuasense.com/mcp`: Docker Desktop's MCP Toolkit, VS Code, Cursor, Windsurf and others.

## Or just the prompt

Paste this into your coding agent in your project:

```text
Add NuaSense field and market data to this project. Docs: https://api.nuasense.com/index.md

1. If `npx skills add nuasense/skills` works here, run it and read the installed skill; otherwise read the docs page above. The full field list is at https://api.nuasense.com/llms-full.txt.
2. A key needs an account at https://platform.nuasense.com with a confirmed email and a verified phone. If I have no key yet, tell me the four steps (sign up, confirm the email, verify the phone under API Keys, create a key) and wait until I paste the key. Read it from an environment variable; never write it into the repository.
3. Call GET https://api.nuasense.com/api/nuasense/v1/catalog with the key. It lists the datasets this key reaches. Build only against what it lists.
4. Send the header `X-NuaSense-Client: skill/0.1.0; agent=<your name>` on every call, with your own name as the agent.
5. Pick the dataset for the task, make one call with a small `limit`, show me the rows, then build the integration. On 429 wait for `Retry-After`; on 402 tell me what needs credit and stop.
```

## What is in this repository

| Path | What |
|---|---|
| `skills/nuasense/SKILL.md` | the skill |
| `.claude-plugin/plugin.json` | the Claude Code plugin manifest (this repository is the plugin) |
| `.claude-plugin/marketplace.json` | the marketplace listing that points at it |
| `.mcp.json` | the hosted MCP server the plugin adds |
| `gemini-extension.json` | the Gemini CLI extension manifest (skill and MCP server) |

Docs as Markdown: https://api.nuasense.com/llms.txt (index), https://api.nuasense.com/index.md (everything on one page), https://api.nuasense.com/llms-full.txt (every field). The HTML reference with a call to paste per dataset is in the console at https://platform.nuasense.com/docs.

## Your data

When you connect a station, NuaSense stores its readings and computes your forecast, spray windows and risks from them. The readings are used aggregated and anonymised to improve forecasts and models for everyone; they are never passed on individually without your opt-in, and positions on public maps are blurred. You can disconnect the station at any time and have all its data deleted.

MIT licence. Issues: https://github.com/nuasense/skills/issues · info@nuasense.com

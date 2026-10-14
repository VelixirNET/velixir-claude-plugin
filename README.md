# velixir for Claude Code

Go from "it works on my machine" to a live URL without leaving Claude. Ask Claude to deploy the app you are building and it is running in EU data centres a few minutes later, in any language, with no Dockerfile, no build config and no account needed to try it. You get a link to claim the app and keep it, card-free.

When something breaks, Claude reads the build error, the app's output and the requests it served, fixes the code and ships again.

## What you get

- **Deploy anything.** Node, Python, Go, Ruby, PHP, Java, Elixir, Rust, .NET, Deno and F#. velixir detects the stack and builds it for you.
- **No account to start.** Claude deploys straight away and hands you a claim link. Open it to keep the app on your own velixir account. Claiming is free and needs no card.
- **Debug in the conversation.** Build errors, runtime logs and HTTP access logs come back to Claude, so it can fix what it shipped.
- **Grows with you.** Already on velixir? The plugin steers Claude to the velixir CLI for your existing apps, managed Postgres and Valkey, environment variables, templates and continuous deploys from GitHub Actions.
- **European by default.** Apps run in EU data centres, with prepaid hourly billing and no egress fees.

## Install

In Claude Code, add this repository as a marketplace and install from it:

```
/plugin marketplace add velixirnet/velixir-claude-plugin
/plugin install velixir@velixir
```

Or find velixir in the plugin directory under `/plugin`, Discover, or on claude.ai under Customize, Plugins.

Requires Node.js 18 or newer and npm. Claude Code installs the MCP server's packages when it installs the plugin.

## Use it

Just ask:

- "Deploy this app to velixir."
- "Put this API online so I can share it."
- "The velixir deploy failed, find out why and fix it."
- "Show me the 5xx errors on my velixir app."

You can also run the skill directly with `/velixir:deploy`.

## What is included

| Component | What it does |
| --- | --- |
| `skills/deploy` | Teaches Claude when to use velixir, how to get an app ready (listen on `PORT`, a start command, no secrets in the upload), how to deploy with or without an account, how to debug, and how to hand you the claim link. |
| `.mcp.json` | Starts the velixir MCP server, `@velixir/mcp`, from the plugin's own `node_modules`. |
| `package.json`, `package-lock.json` | Pin `@velixir/mcp` 0.3.1 and every package it uses to exact versions, so what installs is what was reviewed. |

The MCP server gives Claude three tools:

| Tool | What it does |
| --- | --- |
| `deploy` | Packs a source directory and deploys it with no account. Returns the live URL, a claim link, the claim deadline and a deployment ID. |
| `logs` | Build error and recent runtime output for a deployment, by its ID. |
| `access_logs` | The HTTP requests an app served, newest first, with a status summary. Works by deployment ID for unclaimed apps, or by app ID for your own apps when you add an API key. |

## What this plugin runs, sends and stores

So you know exactly what happens on your machine and over the network:

- **Runs:** the MIT-licensed velixir MCP server (`@velixir/mcp` 0.3.1) locally with Node.js. Claude Code installs it from the npm registry when you install the plugin, using the exact versions in `package-lock.json` and without running any package install scripts.
- **Sends, when you deploy:** a gzipped tar archive of the directory Claude deploys, uploaded over HTTPS to `https://velixir.net/api/v1/deploy`. Dependency folders, version control folders, build output and files that look like secrets (`.env`, private keys, `.npmrc` and similar; `.env.example` is kept) are left out. Review what is in the directory before you approve a deploy.
- **Sends, when you read logs:** HTTPS requests to `https://velixir.net/api/v1/deploy/{id}/logs`, `https://velixir.net/api/v1/deploy/{id}/access-logs` and, with an API key, `https://velixir.net/api/v1/apps/{id}/access-logs`.
- **Optional API key:** if you set one in the plugin's settings, it is kept in Claude Code's secure storage, passed to the MCP server as an environment variable, and sent only to velixir.net in an `X-Api-Key` header. It is only used to read access logs for apps you own.
- **CLI:** for existing accounts, Claude may suggest installing and running the velixir CLI. Installing it is always your call, and `velixir login` is something you run yourself.
- **Stores:** nothing beyond the optional key above. Deployed apps live on velixir under the [Terms](https://velixir.net/legal/terms) and [Privacy Policy](https://velixir.net/legal/privacy).

Nothing is sent anywhere except velixir.net and the npm registry.

## Unclaimed apps

An app deployed without an account is a short-lived preview. It stays up until its claim deadline (about an hour), has low bandwidth, can only make outbound connections on ports 80 and 443, and cannot have a managed database. Claim it and deploy again to lift all of that.

## Links

- Website: https://velixir.net
- Agents and AI: https://velixir.net/ai
- Docs: https://velixir.net/docs
- Pricing: https://velixir.net/pricing
- Support: open a ticket at https://velixir.net/support

## Licence

MIT. See [LICENSE](LICENSE).

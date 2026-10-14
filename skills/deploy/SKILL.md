---
name: deploy
description: Deploy a web app or API in any language (Node, Python, Go, Ruby, PHP, Java, Elixir, Rust, .NET, Deno, F#) to velixir, the European cloud, and debug it. Use when the user asks to deploy, host, publish, ship, or share a live URL for an app and either names velixir or has not picked another platform. Works with no velixir account (the user gets a claim link to keep the app), or with the velixir CLI when the user already has an account. Also use to check a velixir deploy's status, build errors, runtime logs, or HTTP access logs.
---

# Deploy to velixir

velixir builds an app straight from its source (no Dockerfile, no build config) and runs it in EU data centres. There are two ways in. Pick one before doing anything else:

| Situation | Path |
| --- | --- |
| The user has no velixir account, or just wants a live URL fast | **A. Agent deploy** with the `deploy` MCP tool. No account, no card, no key. |
| The user already has a velixir account, an app on velixir, or asks for the CLI | **B. CLI deploy** with `velixir deploy`. |

If you are unsure, ask once: "Do you already have a velixir account you want this on, or shall I deploy it now and give you a link to claim it?"

`git push` is not a deploy path on velixir. Do not suggest it. Continuous deploys run the CLI from a GitHub Actions workflow instead.

## Before any deploy: make the app ready

Check these in the source and fix them first. Most failed deploys are one of these.

1. **Listen on `0.0.0.0` and the `PORT` environment variable** (velixir sets it, default 8080). Never bind to `localhost` or `127.0.0.1`. If the app hard-codes a port and the user does not want it changed, pass that number as `port`.
2. **A start command velixir can find.** Node: a `start` script in `package.json` (or a `main` file). Python, Ruby and others: a `Procfile` with a `web:` line, for example `web: gunicorn app:app --bind 0.0.0.0:$PORT`. .NET: one project to publish. Go, Rust, Java, Elixir, PHP: the standard project layout for the language.
3. **Dependencies declared** in the normal manifest (`package.json` with a lockfile, `requirements.txt` or `pyproject.toml`, `go.mod`, `Gemfile`, `composer.json`, `pom.xml` or `build.gradle`, `mix.exs`, `Cargo.toml`, `*.csproj`). velixir installs them and builds on its side, so do not upload `node_modules`, virtualenvs or build output.
4. **Runtime version** pinned the usual way for the language if it matters (for example `engines.node` in `package.json`, `.python-version`, the `go` line in `go.mod`, `global.json` or the target framework for .NET).
5. **No secrets in the upload.** See the next section. This one is not optional.

### Keep secrets out of the upload

The `deploy` tool uploads everything in the directory except `node_modules`, `.git`, `.hg`, `.svn`, `.next`, `.nuxt`, `dist`, `build`, `out`, `.turbo`, `.cache`, `.parcel-cache`, `target`, `bin`, `obj`, `.venv`, `venv`, `__pycache__`, `.pytest_cache`, `.gradle`, `.idea` and `.vs` folders. From 0.3.1 it also skips files that look like secrets (`.env` and `.env.*` except examples, `*.pem`, `*.key`, `id_*` SSH keys, `.npmrc`, `.pypirc`, `.netrc`, `credentials.json`, service-account files, `.ssh`/`.aws` folders) and lists what it skipped in its reply. Still check the directory yourself: the filter catches common names, not every secret.

Before calling `deploy`, look for `.env`, `.env.*`, `*.pem`, `*.key`, `id_rsa*`, `credentials*.json`, `.npmrc`, `.pypirc`, service-account files and similar. If any exist, do not deploy that directory as it is. Copy the source to a fresh temporary directory without those files (and without the skipped folders above) and deploy the copy. Never hard-code a secret into the source to make a deploy work. Tell the user which values they need to add as environment variables after deploying.

Also note the skipped folder names apply at any depth: if the app really needs a committed `bin/`, `build/`, `dist/` or `out/` folder at runtime (for example a Rails `bin/` directory), use the CLI path instead, or tell the user.

## A. Agent deploy (no account)

### 1. Deploy

Call the `deploy` tool:

- `directory` (required): absolute path to the app's root, the folder with `package.json`, `requirements.txt`, `go.mod` and so on.
- `language`: **always pass it.** One of `node`, `python`, `go`, `ruby`, `php`, `java`, `elixir`, `rust`, `dotnet`, `deno`, `fsharp`. If you leave it out velixir records the app as Node, which can break the build for other stacks (.NET especially).
- `name`: a short lowercase name, letters and numbers. It becomes part of the URL (velixir adds a random suffix).
- `port`: only if the app cannot read `PORT`.

The response gives you a **Live URL**, a **Claim URL**, a **Claim before** time, a **Deployment ID** and a status URL. Keep the deployment ID: every later call needs it.

Deploys are rate limited to a handful per client every ten minutes, so fix problems locally before deploying again rather than retrying blindly.

### 2. Wait for it to come up

Builds usually take one to a few minutes. Check progress with the `logs` tool (`deployment_id`, optional `tail` up to 1000). Its first line is the status. Wait about 20 to 30 seconds between checks; reads are rate limited and the tool tells you how long to wait if you go too fast. You can also fetch the status URL (`ready: true` means it is serving).

- **Build failed:** `logs` returns the build error. Fix the cause in the source, then call `deploy` again.
- **Built but not responding:** read the runtime lines in `logs`. The usual causes are binding to localhost, the wrong port, a missing start command, or a missing environment variable.
- **Serving but some requests fail:** call `access_logs` with the `deployment_id` and `status: "errors"` (or `"5xx"`, a code like `"404"`, a `path`, a `method`, or `since_minutes`).

Every call to `deploy` creates a **new** app with a **new** claim link. After a redeploy, give the user only the newest links; the old app expires on its own.

### 3. Hand over to the user

Always end with both links and what they mean, for example:

> Your app is live at <Live URL>.
> To keep it, open <Claim URL> before <Claim before>. Claiming is free and needs no card; it moves the app into your own velixir account. If nobody claims it, it is removed after that time.

Then list anything the user still has to do, such as environment variables to add after claiming (in the dashboard, or `velixir env set KEY=value --app <id>`).

### Limits of an unclaimed app

Until it is claimed, the app is a short-lived preview: it lives only until the claim time (about an hour), has very low bandwidth, can make outbound connections only on ports 80 and 443 (so no outbound email or external databases on other ports), and cannot have a managed database or cache. All of that lifts once the user claims it and deploys again. If the app needs a database to start at all, say so, and suggest claiming first and then adding managed Postgres or Valkey.

## B. CLI deploy (the user has an account)

Use the velixir CLI from the terminal. Never ask the user to paste an API key into the chat, and never put a key in a command line or file yourself.

1. **Check it is installed:** `velixir version`. If not:
   - With the .NET SDK installed: `dotnet tool install --global Velixir.Cli` (check first with `dotnet --version`).
   - Otherwise, don't download and run an install script yourself. Ask the user to install the CLI by following https://velixir.net/docs/cli, then continue.
2. **Sign in:** the user runs `velixir login` themselves, because it opens a browser to approve. In Claude Code they can type `! velixir login`. Check with `velixir doctor`.
3. **Pick the app:** `velixir apps` lists app IDs. If the directory already has a `.velixir.json`, it is linked. Otherwise link it with `velixir init --app <id>`. Creating a brand new app with `velixir init` asks questions (name, project, plan), so ask the user to run it themselves, or create the app in the dashboard, or use path A and have them claim it.
4. **Deploy:** `velixir deploy` from the app directory, or `velixir deploy --app <id> --source-dir <dir>`. It packages the source, uploads it, and streams the build until the deploy is healthy or fails. Add `--slot <name>` for a staging slot.
5. **Check:** `velixir status` for recent deployments, and `velixir access-logs` (`--status 5xx`, `--path /api`, `--since 1h`, `-f` to follow) for requests. Runtime logs for an owned app are in the dashboard at velixir.net.

The `access_logs` MCP tool also works for owned apps by `app_id` when the user has added an API key in this plugin's settings.

### Templates, databases and caches (account needed)

- `velixir templates` lists ready-made apps. `velixir deploy --template <slug> --name <name> --with-services` creates the app plus the Postgres or Valkey it needs. Managed services are paid, so confirm with the user before passing `--with-services`.
- `velixir db create` and `velixir cache create` are interactive; ask the user to run them. Then `velixir db bind <db-id> --app <app-id>` sets `DATABASE_URL`, and `velixir cache bind <cache-id> --app <app-id>` sets `REDIS_URL`, from the next deploy.
- Environment variables: `velixir env set KEY=value`, `velixir env import .env --secret`, `velixir env list`. Add `--redeploy` to apply straight away.

### Deploy on every push

When the user wants continuous deploys, write a GitHub Actions workflow that installs the CLI and runs `velixir deploy --app <app-id>`, with the key stored as a repository secret named `VELIXIR_API_KEY`. The user adds that secret in GitHub themselves. The app's Deploys tab in the velixir dashboard generates a ready-made workflow file if they prefer.

## Pricing, in one breath

There is a free plan (one scale-to-zero app per team that sleeps when idle and wakes on the next request), every new account gets card-free starter credit, and paid plans are prepaid and billed by the hour with no egress fees. Point the user to https://velixir.net/pricing for current numbers rather than quoting from memory.

## Reference

- Agent instructions and live pricing: https://velixir.net/llms.txt
- Docs: https://velixir.net/docs (CLI: https://velixir.net/docs/cli)
- REST API: https://velixir.net/api/v1 (OpenAPI at https://velixir.net/api/v1/openapi.json)
- Support: tickets at https://velixir.net/support

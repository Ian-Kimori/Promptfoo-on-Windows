# Promptfoo-on-Windows

The main thing to check first is Node.js. Promptfoo requires Node.js 22.22.0 or newer, and Node.js 24 LTS is recommended.

**1. Install Node.js 24 LTS** (skip if `node --version` already shows 22.22+)

In PowerShell:

```powershell
winget install OpenJS.NodeJS.LTS
```

Or download the installer from nodejs.org. If you juggle Node versions, use nvm-windows instead:

```powershell
nvm install 24
nvm use 24
```

Close and reopen your terminal, then confirm:

```powershell
node --version
```

**2. Install promptfoo globally**

```powershell
npm install -g promptfoo
promptfoo --version
```

Alternatively, skip the global install and run it on demand with `npx promptfoo@latest`, which always pulls the newest version.

**3. Start a project**

```powershell
promptfoo init
```

This walks you through creating a promptfooconfig.yaml file.

**Common Windows snag:** if PowerShell says running scripts is disabled when you call `promptfoo`, that's the execution policy blocking npm's `.ps1` shim. Either run `promptfoo.cmd` instead, or allow local scripts for your user:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

If `promptfoo` isn't found at all after install, run `where promptfoo` to check it's on your PATH (npm's global folder is usually `%APPDATA%\npm`).

Two separate things are going on here, and neither is a problem with your install.

**1. You ran the wrong command.** `promptfoo init` created a *red team* config, which starts with no test cases because they're generated from your plugins and strategies. `promptfoo eval` only runs tests that already exist, hence the warning. For red teaming, use:

```powershell
promptfoo redteam run
```

That generates the attack test cases and then evaluates them in one go. (You can split it into `promptfoo redteam generate` followed by `promptfoo redteam eval` if you want to inspect the generated tests first.)

**2. The OpenAI key isn't set, and the suggested fix is Unix syntax.** `export` doesn't work in PowerShell. Use one of these instead:

For the current PowerShell session only:
```powershell
$env:OPENAI_API_KEY = "sk-..."
```

Persistently for your user (open a new terminal afterwards):
```powershell
setx OPENAI_API_KEY "sk-..."
```

Or put it in a `.env` file next to `promptfooconfig.yaml`:
```
OPENAI_API_KEY=sk-...
```
and run `promptfoo redteam run --env-file .env`. If you go this route, keep `.env` out of version control.

**Check your target, too.** The error shows your target is `openai:gpt-5.6`, labelled with your app descriptions (web portals, REST API, DB, etc.). That means promptfoo will attack the raw OpenAI model, not your actual application. If you want to test your own LLM-backed service, change the `targets` section in `promptfooconfig.yaml` to point at it, typically an HTTP provider (`id: https`) with your endpoint URL, request body template, and a `transformResponse` that pulls out the reply text. If testing the bare model is intentional, the key alone is enough.

The `telemetry.shutdown() timed out` line is harmless, usually caused by a proxy or firewall blocking telemetry. To silence it, set `PROMPTFOO_DISABLE_TELEMETRY=1` the same way you set the API key.

Same idea, just swap the provider and key.

**1. Set the Anthropic key** (PowerShell):

```powershell
$env:ANTHROPIC_API_KEY = "sk-ant-..."
```

Or `setx ANTHROPIC_API_KEY "sk-ant-..."` for persistence, or `ANTHROPIC_API_KEY=sk-ant-...` in your `.env` file.

**2. Point the target at Claude** in `promptfooconfig.yaml`:

```yaml
targets:
  - id: anthropic:messages:claude-sonnet-5-5
    label: web portals, mobile apps, graphql, rest api, DB, OS, codebase
```

Swap in `claude-opus-5-5` or `claude-haiku-5-5` if you prefer (Haiku is cheapest for big red team runs).

**3. Make the attacker and grader use Claude too.** Red teaming involves more than the target: a model generates attack prompts and another grades the responses. If those default to OpenAI you'll hit the same missing-key error again, so set them explicitly:

```yaml
redteam:
  provider: anthropic:messages:claude-sonnet-5-5
  # ...your existing purpose, plugins, strategies

defaultTest:
  options:
    provider: anthropic:messages:claude-sonnet-5-5
```

Then run:

```powershell
promptfoo redteam run
```

One practical caveat: using Claude as the *attacker* model means some harmful-content plugins may get fewer or softer test cases, since Claude will decline to write certain attack prompts. Promptfoo routes many of those plugins through its own remote generation service by default, so you'll mostly be fine, but if a category comes back thin, that's likely why.

As before, if you actually want to test your own application rather than Claude itself, the target should be your app's HTTP endpoint, with Claude only in the `redteam.provider` and grader slots.

+++
title = "Configuring Claude Code with DeepSeek V4 PRO APIs"
date = 2026-05-10
draft = false
[taxonomies]
  tags = ["C++", "Tools"]
[extra]
  toc = true
  keywords = "Claude Code, DeepSeek, AI, Dev Container"
+++

## Motivation

[PawnDB](https://github.com/xiahualiu/PawnDB) is a C++17 in-memory database. Modern C++ involves a lot of boilerplate and repetitive patterns. AI coding assistants can absorb that, but the major options are proprietary, expensive, or tied to specific models. DeepSeek V4 PRO provides a capable, cost-effective alternative — and Claude Code supports it through an Anthropic-compatible API endpoint.

This post walks through how PawnDB's dev container is set up to use DeepSeek V4 PRO as the backend for **Claude Code**, so anyone can reproduce the same setup.

## Prerequisites: Dev Container with Node.js

Claude Code is a VS Code extension that runs on Node.js. A base Ubuntu image doesn't include it, so the Dockerfile installs it explicitly:

```dockerfile
RUN apt-get update \
    && apt-get install -y curl \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

The dev container JSON installs the extension and wires the API key:

```json
{
  "customizations": {
    "vscode": {
      "extensions": [
        "anthropic.claude-code"
      ],
      "settings": {
        "claude-code.primaryApiKey": "${env:ANTHROPIC_AUTH_TOKEN}"
      }
    }
  },
  "runArgs": [
    "--env-file=${localWorkspaceFolder}/.devcontainer/.env"
  ]
}
```

The `.env` file (gitignored) supplies the API key:

```bash
ANTHROPIC_AUTH_TOKEN=<your-deepseek-api-key>
```

## Claude Code Configuration

Claude Code natively targets the Anthropic API. DeepSeek exposes an Anthropic-compatible endpoint that accepts the same request/response shapes, so you can redirect Claude Code by changing the base URL, the model name, and setting a few environment variables.

PawnDB's [`.claude/settings.json`](https://github.com/xiahualiu/PawnDB/blob/main/.claude/settings.json):

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_EFFORT_LEVEL": "max"
  }
}
```

Alternatively, if you're migrating an existing Claude Code installation without a settings file, you can export these as shell environment variables:

```bash
export ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
export ANTHROPIC_AUTH_TOKEN=<your DeepSeek API Key>
export ANTHROPIC_MODEL=deepseek-v4-pro[1m]
export ANTHROPIC_DEFAULT_OPUS_MODEL=deepseek-v4-pro[1m]
export ANTHROPIC_DEFAULT_SONNET_MODEL=deepseek-v4-pro[1m]
export ANTHROPIC_DEFAULT_HAIKU_MODEL=deepseek-v4-flash
export CLAUDE_CODE_SUBAGENT_MODEL=deepseek-v4-flash
export CLAUDE_CODE_EFFORT_LEVEL=max
```

### Key details

**Base URL** — `ANTHROPIC_BASE_URL` points to `https://api.deepseek.com/anthropic`. This is the DeepSeek endpoint that speaks the Anthropic Messages API. Claude Code sends all requests here instead of `https://api.anthropic.com`.

**Model mapping** — Claude Code has several "slots" for different model sizes. These are all mapped to DeepSeek equivalents:

| Slot | Anthropic Default | DeepSeek Mapping |
|------|------------------|-----------------|
| Opus | `claude-opus-4-7` | `deepseek-v4-pro[1m]` |
| Sonnet | `claude-sonnet-4-6` | `deepseek-v4-pro[1m]` |
| Haiku | `claude-haiku-4-5` | `deepseek-v4-flash` |
| Subagent | varies | `deepseek-v4-flash` |

Haiku and subagent models point to `deepseek-v4-flash` for quick, cheap tasks (file reads, simple edits). Opus, Sonnet, and the primary model all use `deepseek-v4-pro[1m]` for heavier reasoning. The `[1m]` suffix enables the 1-million-token context window.

**Effort level** — `CLAUDE_CODE_EFFORT_LEVEL=max` sets maximum reasoning depth for the primary model. This lets `deepseek-v4-pro` spend more thinking tokens on complex refactors and multi-file changes.

## How It Fits Together

From a cold start:

1. **Build the image** — `docker build` runs the Dockerfile, installing Node.js alongside the C++ toolchain.
2. **Open in dev container** — VS Code starts the container, mounts the repo volume, and loads the `.env` file.
3. **Extension activates** — Claude Code reads `.claude/settings.json` and redirects all chat/agent traffic to `api.deepseek.com/anthropic`.
4. **Workflow** — You chat with Claude Code for complex tasks (refactoring, generating tests, explaining code), all powered by DeepSeek's models.

All of this is committed to the repo (except the `.env` secrets), so anyone who clones PawnDB and provides their own DeepSeek API key gets the same setup.

## Trade-offs

**Why DeepSeek over Anthropic's own models?** Cost and access. DeepSeek V4 PRO is substantially cheaper per token while being competitive on reasoning-heavy C++ tasks. It also avoids the need for a separate Anthropic billing account.

**What you lose.** DeepSeek's Anthropic-compatible endpoint doesn't implement every Claude feature. Sometimes the agent gets interrupted and you have to prompt again to continue. Prompt caching and some advanced tool-use patterns may behave differently than on Anthropic's native API.

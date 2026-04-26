+++
title = "Configuring Claude Code and Continue with DeepSeek V4 PRO APIs"
date = 2026-04-26
draft = false
[taxonomies]
  tags = ["C++", "Tools"]
[extra]
  toc = true
  keywords = "Claude Code, Continue, DeepSeek, AI, Dev Container"
+++

## Motivation

[PawnDB](https://github.com/xiahualiu/PawnDB) is a C++17 in-memory database. Modern C++ involves a lot of boilerplate and repetitive patterns. AI coding assistants can absorb that, but the major options are proprietary, expensive, or tied to specific models. DeepSeek V4 PRO provides a capable, cost-effective alternative — and both Claude Code and Continue support it through standard API-compatible endpoints.

This post walks through how PawnDB's dev container is set up to use DeepSeek V4 PRO as the backend for **Claude Code** (chat/agent) and **Continue** (autocomplete), so anyone can reproduce the same setup.

## Architecture Overview

```
┌──────────────────────────────────────────┐
│  VS Code Dev Container                   │
│                                          │
│  ┌──────────────┐  ┌──────────────────┐  │
│  │ Claude Code  │  │ Continue         │  │
│  │ (chat/agent) │  │ (autocomplete)   │  │
│  └──────┬───────┘  └────────┬─────────┘  │
│         │                   │            │
│         │ Anthropic API     │ OpenAI API │
│         │ compat            │ compat     │
└─────────┼───────────────────┼────────────┘
          │                   │
     ┌────▼───────────────────▼────┐
     │  api.deepseek.com           │
     │  /anthropic  │  /beta       │
     │  (messages)  │  (FIM)       │
     └─────────────────────────────┘
```

Two tools, two DeepSeek API surfaces:
- **Claude Code** speaks the Anthropic Messages API — DeepSeek serves a compatible endpoint at `/anthropic`.
- **Continue** uses the OpenAI-compatible endpoint at `/beta` for fill-in-the-middle (FIM) autocomplete.

Both share the same API key, provided through the dev container's `.env` file.

## Prerequisites: Dev Container with Node.js

Claude Code and Continue are VS Code extensions that run on Node.js. A base Ubuntu image doesn't include it, so the Dockerfile installs it explicitly:

```dockerfile
RUN apt-get update \
    && apt-get install -y curl \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

The dev container JSON installs both extensions and wires their API keys:

```json
{
  "customizations": {
    "vscode": {
      "extensions": [
        "anthropic.claude-code",
        "Continue.continue"
      ],
      "settings": {
        "claude-code.primaryApiKey": "${env:ANTHROPIC_AUTH_TOKEN}",
        "continue.apiKey": "${env:CONTINUE_API_KEY}"
      }
    }
  },
  "runArgs": [
    "--env-file=${localWorkspaceFolder}/.devcontainer/.env"
  ]
}
```

The `.env` file (gitignored) supplies a single key for both tools:

```bash
ANTHROPIC_AUTH_TOKEN=<your-deepseek-api-key>
CONTINUE_API_KEY=<your-deepseek-api-key>
```

## Claude Code Configuration

Claude Code natively targets the Anthropic API. DeepSeek exposes an Anthropic-compatible endpoint that accepts the same request/response shapes, so you can redirect Claude Code by changing three things: the base URL, the model name, and the API key.

PawnDB's [`.claude/settings.json`](https://github.com/xiahualiu/PawnDB/blob/main/.claude/settings.json):

```json
{
  "primaryApiKey": "any",
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_SMALL_FAST_MODEL": "deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro"
  },
  "models": {
    "deepseek-v4-pro": {
      "reasoning_effort": "max"
    },
    "deepseek-v4-flash": {
      "reasoning_effort": "high"
    }
  }
}
```

### Key details

**`primaryApiKey: "any"`** — DeepSeek's Anthropic endpoint doesn't validate the `x-api-key` header the same way Anthropic does. The actual authentication is handled by the `ANTHROPIC_AUTH_TOKEN` environment variable set in `.env`, which Claude Code picks up from the dev container. The `primaryApiKey` just needs to be non-empty.

**Base URL** — `ANTHROPIC_BASE_URL` points to `https://api.deepseek.com/anthropic`. This is the DeepSeek endpoint that speaks the Anthropic Messages API. Claude Code sends all requests here instead of `https://api.anthropic.com`.

**Model mapping** — Claude Code has several "slots" for different model sizes. These are all mapped to DeepSeek equivalents:

| Slot | Anthropic Default | DeepSeek Mapping |
|------|------------------|-----------------|
| Opus | `claude-opus-4-7` | `deepseek-v4-pro` |
| Sonnet | `claude-sonnet-4-6` | `deepseek-v4-pro` |
| Haiku | `claude-haiku-4-5` | `deepseek-v4-flash` |
| Small fast | varies | `deepseek-v4-flash` |

Haiku and the "small fast" model point to `deepseek-v4-flash` for quick, cheap tasks (file reads, simple edits). Sonnet and Opus both use `deepseek-v4-pro` for heavier reasoning.

**Reasoning effort** — The `models` block sets `reasoning_effort` per model. `deepseek-v4-pro` gets `"max"` (extended thinking for complex refactors), while `deepseek-v4-flash` gets `"high"` (enough for straightforward completions without burning extra tokens).

## Continue Configuration (Autocomplete)

Continue handles FIM autocomplete — the inline suggestions that appear as you type. Unlike Claude Code (which uses the Anthropic protocol), Continue's autocomplete path talks to providers through an OpenAI-compatible interface.

PawnDB's [`.continue/config.yaml`](https://github.com/xiahualiu/PawnDB/blob/main/.continue/config.yaml):

```yaml
name: DeepSeek V4 Config
version: 1.0.0
schema: v1

models:
  - name: DeepSeek V4 Pro FIM
    provider: openai
    model: deepseek-v4-pro
    apiBase: https://api.deepseek.com/beta
    roles:
      - autocomplete
```

The provider is set to `openai` because Continue uses the OpenAI chat completions format for FIM requests. DeepSeek's `/beta` endpoint supports this, including the FIM-specific tokens that let the model insert code in the middle of a partially typed line.

## How It Fits Together

From a cold start:

1. **Build the image** — `docker build` runs the Dockerfile, installing Node.js alongside the C++ toolchain.
2. **Open in dev container** — VS Code starts the container, mounts the repo volume, and loads the `.env` file.
3. **Extensions activate** — Claude Code reads `.claude/settings.json` and redirects all chat/agent traffic to `api.deepseek.com/anthropic`. Continue reads `.continue/config.yaml` and registers the autocomplete provider.
4. **Workflow** — You chat with Claude Code for complex tasks (refactoring, generating tests, explaining code) and get inline FIM suggestions from Continue as you type new C++.

All of this is committed to the repo (except the `.env` secrets), so anyone who clones PawnDB and provides their own DeepSeek API key gets the same setup.

## Trade-offs

**Why DeepSeek over Anthropic's own models?** Cost and access. DeepSeek V4 PRO is substantially cheaper (>1000x cheaper with KV cache) per token while being competitive on reasoning-heavy C++ tasks. It also avoids the need for a separate Anthropic billing account.

**Why both Claude Code and Continue?** They solve different problems. Claude Code is an agent — it reads files, runs commands, makes multi-file edits. Continue is a keystroke-level autocomplete that predicts what you're about to type. Using DeepSeek for both means one API key, one bill, two tools.

**What you lose.** DeepSeek's Anthropic-compatible endpoint doesn't implement every Claude feature. Sometimes the agent gets interrupted and you have to prompt again to continue.
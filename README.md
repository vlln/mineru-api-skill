<h1 align="center">mineru-api-skill</h1>

<p align="center">
  <strong>Parse PDFs into structured markdown via MinerU API — no local GPU required.</strong><br/>
  Agent Skills for interacting with a remote MinerU API server. Extracts text, tables, formulas, and images from PDFs using sync/async parsing, batch processing, and multiple backend engines.
</p>

<p align="center">
  <a href="https://github.com/vlln/mineru-api-skill/stargazers"><img src="https://badgen.net/github/stars/vlln/mineru-api-skill?label=%E2%98%85" alt="GitHub stars" /></a>
  <img src="https://badgen.net/badge/license/MIT/blue" alt="MIT" />
  <img src="https://badgen.net/badge/spec/Agent%20Skills/8257D0" alt="Agent Skills spec" />
</p>

---

## Installation

### [skit](https://github.com/vlln/skit) (Recommended)

```bash
skit install https://github.com/vlln/mineru-api-skill/tree/main/skills/mineru-api
```

### Manually

| Agent | Command |
|-------|---------|
| **Claude Code** | `cp -r skills/mineru-api .claude/skills/` |
| **Codex** | `cp -r skills/mineru-api ~/.codex/skills/` |
| **Kimi** | `cp -r skills/mineru-api ~/.kimi/skills/` |

---

## Skills

| Skill | Description |
|-------|-------------|
| [mineru-api](skills/mineru-api/SKILL.md) | Parse PDF documents via a MinerU API server — sync and async submission, backend selection, batch processing, and result extraction. |

## Requirements

- A running MinerU API server (self-hosted, transient CLI, or official cloud)
- Python 3.8+ (standard library only, no pip install required)

## Background

[MinerU](https://github.com/opendatalab/MinerU/tree/master) is an open-source
PDF-to-Markdown parser by OpenDataLab. It extracts text, tables, formulas, and
images from PDFs using a hybrid pipeline of ML models and heuristics. It
supports multiple backends (pipeline, hybrid-auto-engine, VLM), OCR for scanned
documents, and multi-language recognition.

**This project wraps MinerU's HTTP API so that Agent Skills-compatible agents can
parse PDFs without running MinerU locally.**

## MinerU API — Configuration Options

There are three ways to provision a MinerU API server:

### 1. Self-Hosted Server

Deploy MinerU's built-in API server on your own GPU machine:

```bash
pip install mineru
mineru-api --host 0.0.0.0 --port 8000
```

Point your `.env` at it:

```bash
MINERU_API_URL=http://your-gpu-server:8000
```

Best for teams with dedicated GPU infrastructure. Full control over queueing,
models, and resources.

### 2. Transient API via `mineru` CLI

The `mineru` CLI can start a transient API server for ad-hoc use:

```bash
mineru serve --port 8000
```

Same `.env` configuration as above. Useful for quick local testing before
deploying a persistent server.

### 3. Official MinerU Cloud API

OpenDataLab provides a hosted MinerU API. Sign up at their platform, obtain an
API key, and set:

```bash
MINERU_API_URL=https://api.mineru.opendatalab.com
```

Refer to [MinerU's documentation](https://github.com/opendatalab/MinerU) for
pricing, rate limits, and API key provisioning.

## How It Works

```
Agent (Claude Code / skill-compatible)
    │
    ├─ uses skill: mineru-api
    │
    └─ scripts/mineru-api  (CLI wrapper, mirrors `mineru` flags)
        │
        ├─ POST /file_parse       sync parse
        ├─ POST /tasks            async submit
        ├─ GET  /tasks/{id}       poll status
        ├─ GET  /tasks/{id}/result  fetch output
        └─ GET  /health           health check
            │
            ▼
    MinerU API Server
        │
        └─ MinerU engine (pipeline / hybrid / VLM backend)
            │
            ▼
    output/
    └── paper/
        └── auto/
            └── paper.md
```

The `scripts/mineru-api` wrapper mirrors the `mineru` CLI flags exactly —
agents that already know the `mineru` interface can use this tool without
relearning anything. The wrapper handles multipart upload, sync/async
submission, polling, and result download transparently.

## License

MIT
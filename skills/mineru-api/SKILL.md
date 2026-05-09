---
name: mineru-api
description: Parse PDF documents via a remote MinerU API server using a CLI wrapper that mirrors the `mineru` interface. Covers single-file and batch PDF parsing, async submission, and health checks. Use when the user wants to extract markdown, tables, and formulas from PDFs through an existing MinerU deployment.
metadata:
  skit:
    version: 0.1.0
    requires:
      env:
        - MINERU_API_URL
    keywords:
      - pdf-parsing
      - ocr
      - markdown
      - mineru
---

# MinerU API

Use this skill when the user needs to parse PDF documents through a MinerU API
server. The wrapper script at `scripts/mineru-api` mirrors the `mineru` CLI.

## When To Use

- "Parse this PDF via the MinerU server"
- "Extract markdown from all PDFs in this directory"
- "Check if the MinerU API is up"
- "Convert this scanned PDF with OCR through MinerU"

## Prerequisites

The script reads configuration from `.env` in the working directory:

```bash
MINERU_API_URL=http://xxx.xxx.xxx.xxx   # required
MINERU_API_TIMEOUT=600                  # optional, default 600
MINERU_API_POLL_INTERVAL=3              # optional, default 3
```

## Workflow

### 1. Parse with the same flags as `mineru`

The wrapper mirrors `mineru`'s CLI and uses only the Python standard library
(zero third-party dependencies). Any agent that knows `mineru` knows this
tool:

```bash
scripts/mineru-api -p paper.pdf -o output -l en -b pipeline

scripts/mineru-api -p scan.pdf -o output -l ch -m ocr -b pipeline

scripts/mineru-api -p paper.pdf -o output -l en -s 2 -e 6
```

### 2. Output layout (identical to `mineru`)

```
output/
└── paper/
    └── auto/
        └── paper.md
```

### 3. Batch parse a directory

When `-p` points to a directory, the script recursively finds all `.pdf` files:

```bash
scripts/mineru-api -p /data/papers/ -o extracted/ -l en
```

### 4. Async mode (fire-and-poll, for large PDFs)

```bash
scripts/mineru-api --async -p large.pdf -o output -l en
```

## Flag Reference (mirrors `mineru`)

| Flag | Default | Notes |
|------|---------|-------|
| `-p`, `--path` | required | PDF file or directory |
| `-o`, `--output` | required | Output directory |
| `-b`, `--backend` | `pipeline` | `pipeline`, `hybrid-auto-engine`, `vlm-http-client` |
| `-m`, `--method` | `auto` | `auto`, `txt`, `ocr` |
| `-l`, `--lang` | `en` | `en`, `ch`, `ch_server`, `japan`, `korean`, etc. |
| `-s`, `--start` | `0` | First page (0-indexed) |
| `-e`, `--end` | `-1` | Last page (-1 = all) |
| `-f`, `--formula` | `true` | Enable formula extraction |
| `-t`, `--table` | `true` | Enable table extraction |
| `--async` | off | Submit as tasks and poll |
| `--check` | — | Health check only |

## Decision Guide

| Situation | Flags |
|-----------|-------|
| Normal PDF, any GPU | `-b pipeline -l en` |
| Scanned PDF (image-based) | `-b pipeline -m ocr -l en` |
| High accuracy, GPU available | `-b hybrid-auto-engine -l en` |
| Chinese PDF | `-l ch` |
| Extract only pages 3–7 | `-s 2 -e 6` |
| Offload to remote GPU | `-b vlm-http-client -u http://gpu-node:30000` |

## Raw API (fallback)

When the script is unavailable or programmatic control is needed:

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/health` | Health check |
| `POST` | `/file_parse` | Sync parse (multipart, blocks until done) |
| `POST` | `/tasks` | Async submit, returns `task_id` |
| `GET` | `/tasks/{id}` | Poll task status |
| `GET` | `/tasks/{id}/result` | Fetch result when status is `done` |

The `/file_parse` endpoint accepts these form fields: `files`, `backend`,
`parse_method`, `lang_list`, `formula_enable`, `table_enable`, `return_md`,
`start_page_id`, `end_page_id`.

## Rules

- For batch jobs, use sync mode — one failure won't block others.
- `.env` must exist in the working directory with `MINERU_API_URL` set.
- If the server is down the user must start it. Run `--check`.

---
name: mineru-api
description: Use this skill when parsing PDF documents through a remote MinerU API server. Covers single-file and batch PDF parsing, async submission, and health checks. Use when the user wants to extract markdown, tables, and formulas from PDFs.
license: MIT
metadata:
  author: vlln
  version: "0.1.0"
requires:
  env:
    - MINERU_API_URL
---

# MinerU API

Parse PDF documents through a remote MinerU API server. Extracts markdown,
tables, and formulas.

## Trigger Keywords

- "mineru", "MinerU"
- "parse PDF", "extract PDF", "PDF to markdown"
- "OCR PDF", "PDF OCR"
- "extract tables from PDF", "extract formulas from PDF"

## Capabilities

- **Single PDF**: Submit a PDF, get markdown output with extracted tables and formulas.
- **Batch**: Process all PDFs in a directory recursively.
- **Async**: Fire-and-forget for large PDFs, poll for results.
- **Health check**: Verify the server is reachable.

## Configuration

Set in `.env` in the working directory:

```bash
MINERU_API_URL=http://xxx.xxx.xxx.xxx   # required
MINERU_API_TIMEOUT=600                  # optional, default 600
MINERU_API_POLL_INTERVAL=3              # optional, default 3
```

## Usage

All operations use the same flag interface:

```
-p paper.pdf -o output -l en -b pipeline
-p scan.pdf -o output -l ch -m ocr -b pipeline
-p paper.pdf -o output -l en -s 2 -e 6
```

### Single PDF

```
-p paper.pdf -o output -l en -b pipeline
```

Output is always:

```
output/
└── paper/
    └── auto/
        └── paper.md
```

### Batch Directory

When `-p` points to a directory, all `.pdf` files are processed recursively:

```
-p /data/papers/ -o extracted/ -l en
```

### Async Mode

Submit as a task and poll for completion. Use for PDFs larger than 100MB or
when server timeouts are a concern:

```
--async -p large.pdf -o output -l en
```

### Health Check

```
--check
```

## Flag Reference

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

## Gotchas

- The `--start` flag is 0-indexed: page 1 is `-s 0`, pages 3–7 are `-s 2 -e 6`.
- For batch jobs, use sync mode — one failure won't block others.
- The output directory structure is always fixed: `<output>/<basename>/auto/<basename>.md`.
- Async mode is recommended for PDFs larger than 100MB.

## References

- `references/api.md` — Raw HTTP API endpoints and form fields. Read only when the flag interface is unavailable or programmatic control over requests is needed.
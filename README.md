# ABBYY FineParser

**High-fidelity document parser for AI pipelines.**

[![Docker image](https://img.shields.io/badge/docker-abbyy%2Ffineparser-2496ED?logo=docker&logoColor=white)](https://hub.docker.com/r/abbyy/fineparser)
[![Documentation](https://img.shields.io/badge/docs-docs.abbyy.com-FF2038)](https://docs.abbyy.com/fine-parser/getting-started/introduction)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)](./LICENSE)

FineParser is a self-hosted document parser for AI pipelines. It runs as one Docker container on CPU hardware and exposes one REST endpoint: send a document, get back its structure and text as DocLang, JSON, or plain text. No cloud account, no GPU, and no document data leaves your infrastructure.

This repository is the community support channel for FineParser. The product itself ships as a Docker image; [open an issue](https://github.com/abbyy/FineParser/issues) to ask a question, report a bug, or request a feature. See [SUPPORT.md](./SUPPORT.md).

---

## What it is

**High fidelity.** FineParser recovers a document's *structure*, not just its text: tables (including borderless ones), multi-column layouts, headings, reading order, and hierarchy. Handwriting, checkboxes, and images are recognized alongside the text. It is built on the [ABBYY FineReader Engine](https://docs.abbyy.com/fine-reader/engine/introduction) recognition core, packaged for developers who want production-grade parsing without an SDK integration. Fidelity is the default posture — the container starts in `accurate` mode, and `-mode fast` is the opt-out when you need throughput more than structure.

**For AI pipelines.** The output is ready for RAG ingestion, agent workflows, or LLM prompts. [DocLang](https://docs.abbyy.com/fine-parser/concepts/doclang) is the recommended format because it keeps that structure at a fraction of the token cost — a table encoded with DocLang's OTSL scheme needs five structural tokens where the equivalent HTML needs twenty-eight.

**A parser you run.** One container, CPU only, one endpoint, on your own machines or in your own cloud.

### Why teams use it

- Keeps tables, columns, and headings intact, and tells you where each one is on the page.
- Outputs [DocLang](https://docs.abbyy.com/fine-parser/concepts/doclang), a compact format built for LLM input and the one we recommend. JSON and plain text are also available.
- Runs as a single container on CPU hardware, on your own machines or in your own cloud.
- Two [recognition modes](https://docs.abbyy.com/fine-parser/basics/recognition-modes): fast mode for high volumes, high-fidelity mode when accuracy matters most.
- Recognizes more than 200 [languages](https://docs.abbyy.com/fine-parser/basics/languages), including Latin, Cyrillic, CJK, and Arabic scripts.
- Start on the free tier, upgrade to a paid plan as your volume grows, or move to [Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise) for high throughput and offline deployment.

---

## Quickstart

Get an API key, run FineParser with Docker, and parse your first document into DocLang in under five minutes.

### Before you begin

- [Docker](https://docs.docker.com/get-docker/) installed and running
- A PDF or image file to parse
- A FineParser API key. Sign up for a plan at [fineparser.ai](https://www.fineparser.ai/#pricing) and the key is emailed to you. See [API keys and plans](https://docs.abbyy.com/fine-parser/basics/api-keys-and-plans).
- Outbound access to ABBYY's license server. If your network restricts egress, allow the ranges listed under [Network requirements](https://docs.abbyy.com/fine-parser/reference/data-privacy#network-requirements) first.

### 1. Get an API key

Sign up for a plan at [fineparser.ai](https://www.fineparser.ai/#pricing). The Free plan gives you 1,000 pages per month. Your API key is emailed to the address you signed up with.

### 2. Start FineParser

Pass your API key as an environment variable and publish port 8080:

```bash
docker run -p 8080:8080 -e FINEPARSER_API_KEY="<your-api-key>" abbyy/fineparser
```

Leave the container running.

### 3. Parse a document

```bash
curl -X POST http://localhost:8080/parse \
  -F "file=@invoice.pdf" \
  -F "outputType=doclang" \
  -o result.doclang
```

The result is the document's structure and text in DocLang:

```xml
<doclang>
  <heading level="1">Invoice 4471</heading>
  <table>
    <ched/>Item<ched/>Qty<ched/>Amount<nl/>
    <fcel/>Consulting<fcel/>12<fcel/>$4,800.00<nl/>
  </table>
</doclang>
```

Swap `doclang` for `json` or `txt` to change the format.

---

## Two ways to run

### Server mode

Start the container without positional arguments and FineParser runs as an HTTP server. It stays up until you stop it, accepting documents on port 8080 and returning the parsed result in the response body. Use this when your applications or other services need to send documents to FineParser over the network.

```bash
docker run -p 8080:8080 -e FINEPARSER_API_KEY="<your-api-key>" abbyy/fineparser
```

The API has one endpoint, `POST /parse`. Send the document as a `multipart/form-data` upload with a `file` field holding the document and an `outputType` field set to `doclang`, `json`, or `txt`. Each request handles one file.

Parsing is synchronous. The connection stays open while the document is processed, so give your client a timeout that fits the size of the documents you send.

| Status | Cause |
|---|---|
| `200` | Success. The response body is the parsed document. |
| `400` | `outputType` is missing or not one of `doclang`, `json`, or `txt`, or the request does not contain exactly one file. |
| `500` | The container could not write the uploaded file to its working directory. |

See [REST API](https://docs.abbyy.com/fine-parser/concepts/rest-api) for Python and Node.js examples.

### CLI mode

Start the container with an input file and an output file as positional arguments and FineParser runs as a one-shot command instead. The container starts, parses the document, writes the result, and exits. Use this for scripts, batch jobs, and quick checks.

```bash
docker run --rm -v "$PWD:/data" -e FINEPARSER_API_KEY="<your-api-key>" abbyy/fineparser \
  /data/invoice.pdf /data/invoice.doclang
```

Both paths are inside the container, so mount the directory that holds your document and receives the output. There is no `outputType` here — the format comes from the extension of the output file (`.doclang`, `.json`, `.txt`). Flags must come before the positional arguments.

Every run starts a new container and loads the recognition engine before it reads your document, so looping over a folder is slow. If you have more than a handful of documents, run the container as a server and post them to it instead.

See [CLI](https://docs.abbyy.com/fine-parser/concepts/cli).

---

## Output formats

Pass the `outputType` field with each request. There is no default, so every request must name a format. Unlike recognition mode and languages, the format is chosen per request, so one container can serve all three.

| Output type | Best for | Structure | Positions |
|---|---|---|---|
| `doclang` | LLM prompts, RAG, agents | Yes | Yes |
| `json` | Application code, pipelines | Yes | Yes |
| `txt` | Search indexing, plain extraction | No | No |

**DocLang** is an open, AI-native document standard from a working group founded by IBM, NVIDIA, Red Hat, ABBYY, HumanSignal, and Forgis, developed under Joint Development Foundation Projects as an LF AI & Data project. It is a constrained XML format built for LLM tokenizers: every element carries a semantic role, bounding box coordinates, and a position in the reading order, and tables keep their full grid structure. Because it is a specification rather than a product, code written against FineParser's output works with any other DocLang producer.

Learn more at [doclang.ai](https://doclang.ai/) and in the [doclang-project/doclang](https://github.com/doclang-project/doclang) repository. See [Output formats](https://docs.abbyy.com/fine-parser/basics/output-formats).

---

## Configuration

FineParser is configured once, when the container starts, with command-line flags and environment variables. The settings hold for the life of the process.

| Flag | Default | |
|---|---|---|
| `-mode` | `accurate` | `accurate` reads the text and reconstructs layout, tables, checkboxes, and reading order. `fast` reads the text only and skips structure analysis. See [Recognition modes](https://docs.abbyy.com/fine-parser/basics/recognition-modes). |
| `-language` | `English` | Comma-separated list of recognition languages, e.g. `-language "English,German"`. See [Languages](https://docs.abbyy.com/fine-parser/basics/languages). |

```bash
docker run -p 8080:8080 -e FINEPARSER_API_KEY="<your-api-key>" abbyy/fineparser \
  -mode fast -language "English,German"
```

**One configuration per container.** Recognition settings cannot be changed per request. Every document a container receives is processed with the mode and languages it was started with. If different document sets need different settings, run one container per configuration and route each request to the container that matches.

**Pick only the languages your documents actually contain.** Each added language widens the character set the engine considers on every page, which means slower processing, higher memory use, and lower accuracy.

FineParser listens on port 8080 inside the container. Publish that port with `-p HOST:CONTAINER`. `-p 8080:8080` makes the API reachable on host port 8080 from any machine on the network; `-p 127.0.0.1:9000:8080` limits it to localhost on port 9000.

See [Configuration](https://docs.abbyy.com/fine-parser/basics/configuration).

---

## Requirements

- Docker, on CPU hardware. No GPU is required.
- Outbound access to ABBYY's license server:

  | Protocol | Address range |
  |---|---|
  | IPv4 | `185.146.155.0/24` |
  | IPv6 | `2620:122:f003::/48` |

> [!WARNING]
> If FineParser cannot reach these addresses, it does not parse any documents. Requests to `/parse` fail until connectivity is restored. For environments with no outbound connectivity, see [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise).

---

## Plans

FineParser is licensed by pages parsed per month. Your API key gives the container access to your plan's monthly page allowance. Sign up at [fineparser.ai](https://www.fineparser.ai/#pricing).

| Plan | Pages per month | How to get it |
|---|---|---|
| Free | 1,000 | Sign up at fineparser.ai. Valid for one year. Credit card required. |
| Starter | 25,000 | Self-service upgrade |
| Growth | 250,000 | Self-service upgrade |
| Business | 1,000,000 | Self-service upgrade |
| Enterprise | Custom | [Talk to sales](https://docs.abbyy.com/fine-parser/reference/enterprise) |

Processing stops when you reach your plan's monthly page limit. FineParser does not offer pay-as-you-go pricing or overage billing — upgrade to a larger plan and the new allowance is available immediately. For subscription management, upgrades, invoices, and cancellation, see [API keys and plans](https://docs.abbyy.com/fine-parser/basics/api-keys-and-plans).

---

## Privacy

Documents are processed locally and never transmitted to ABBYY or anyone else. Two things do leave the container: license validation, which checks your remaining page balance and reports pages parsed, and telemetry about each job — page count, document type, recognition mode, output format, outcome, duration, license status, and your license key. Telemetry is always on and cannot be disabled.

FineParser never collects document content, recognized text, filenames, file paths, file sizes, customer names, email addresses, account identifiers, user identities, or IP addresses.

See [Data privacy and telemetry](https://docs.abbyy.com/fine-parser/reference/data-privacy).

---

## Using AI coding agents

This repository ships an [`AGENTS.md`](./AGENTS.md) runbook. Point your coding agent at it and it knows how to deploy FineParser, guide you through getting a license key, and run documents through the API. Claude Code, Codex, Cursor, and GitHub Copilot read it automatically when it is in your project root.

You can also connect the ABBYY documentation MCP server so your agent can look up FineParser endpoints, parameters, and output formats while it writes your integration. The server is read-only; it searches public documentation and does not touch your container, your documents, or your API key.

```bash
# Claude Code
claude mcp add abbyy-docs https://docs.abbyy.com/mcp --transport http

# Codex
codex mcp add abbyy-docs --url https://docs.abbyy.com/mcp
```

See [AI coding agents](https://docs.abbyy.com/fine-parser/getting-started/ai-agents).

---

## Documentation

Full documentation lives at **[docs.abbyy.com/fine-parser](https://docs.abbyy.com/fine-parser/getting-started/introduction)**.

| | |
|---|---|
| **Getting started** | |
| [Introduction](https://docs.abbyy.com/fine-parser/getting-started/introduction) | What FineParser is and what it does. |
| [Quickstart](https://docs.abbyy.com/fine-parser/getting-started/quickstart) | Key, container, first parse, in five minutes. |
| [AI coding agents](https://docs.abbyy.com/fine-parser/getting-started/ai-agents) | Connect the docs MCP server to your agent. |
| **Basics** | |
| [Configuration](https://docs.abbyy.com/fine-parser/basics/configuration) | Startup flags, environment variables, ports. |
| [Recognition modes](https://docs.abbyy.com/fine-parser/basics/recognition-modes) | High fidelity vs. fast. |
| [Languages](https://docs.abbyy.com/fine-parser/basics/languages) | The full table of 200+ language keys. |
| [Output formats](https://docs.abbyy.com/fine-parser/basics/output-formats) | DocLang, JSON, and plain text compared. |
| [API keys and plans](https://docs.abbyy.com/fine-parser/basics/api-keys-and-plans) | Registration, allowances, subscriptions. |
| **Concepts** | |
| [DocLang](https://docs.abbyy.com/fine-parser/concepts/doclang) | The AI-native document format, annotated. |
| [REST API](https://docs.abbyy.com/fine-parser/concepts/rest-api) | `POST /parse`, with cURL, Python, and Node.js. |
| [CLI](https://docs.abbyy.com/fine-parser/concepts/cli) | One-shot parsing from the command line. |
| **Reference** | |
| [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise) | Unlimited pages, air-gapped deployment. |
| [Data privacy and telemetry](https://docs.abbyy.com/fine-parser/reference/data-privacy) | What leaves the container, and what never does. |
| [Support and SLAs](https://docs.abbyy.com/fine-parser/reference/support) | Support by plan. |
| **Release notes** | |
| [1.0](https://docs.abbyy.com/fine-parser/release-notes/1.0) | Initial release. |

---

## Support

| Plan | Support |
|---|---|
| Free, Starter, Growth | Community — [this repository](https://github.com/abbyy/FineParser/issues) |
| Business | Community and [ABBYY technical support](https://support.abbyy.com/hc/en-us) |
| Enterprise | Enterprise SLAs and ABBYY technical support |

See [SUPPORT.md](./SUPPORT.md) for what to include in a report, and [CONTRIBUTING.md](./CONTRIBUTING.md) for how to help.

---

## License

The contents of this repository are licensed under [Apache-2.0](./LICENSE). The FineParser Docker image is a commercial ABBYY product, licensed separately under the terms you accept when you sign up for a plan.

<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="./assets/fineparser-logo-dark.svg">
  <source media="(prefers-color-scheme: light)" srcset="./assets/fineparser-logo.svg">
  <img alt="ABBYY FineParser" src="./assets/fineparser-logo.svg" width="420">
</picture>

### High-fidelity document parser for AI pipelines

[![Docker image](https://img.shields.io/badge/docker-abbyyteam%2Ffineparser-2496ED?logo=docker&logoColor=white)](https://hub.docker.com/r/abbyyteam/fineparser)
[![Documentation](https://img.shields.io/badge/docs-docs.abbyy.com-FF2038)](https://docs.abbyy.com/fine-parser/getting-started/introduction)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue)](./LICENSE)

</div>

FineParser is a self-hosted document parser for AI pipelines. It runs as one Docker container on CPU hardware and exposes a small REST API: submit a document, then download its structure and text as DocLang, JSON, or plain text. No cloud account, no GPU, and no document data leaves your infrastructure.

This repository is the community support channel for FineParser. The product itself ships as a Docker image; [open an issue](https://github.com/abbyy/FineParser/issues) to ask a question, report a bug, or request a feature. See [SUPPORT.md](./SUPPORT.md).

---

## What it is

**High fidelity.** FineParser recovers a document's *structure*, not just its text: tables (including borderless ones), multi-column layouts, headings, reading order, and hierarchy. Handwriting, checkboxes, and images are recognized alongside the text. It is built on the [ABBYY FineReader Engine](https://docs.abbyy.com/fine-reader/engine/introduction) recognition core, packaged for developers who want production-grade parsing without an SDK integration. Fidelity is the default posture — the container starts in `accurate` mode, and `-mode fast` is the opt-out when you need throughput more than structure.

**For AI pipelines.** The output is ready for RAG ingestion, agent workflows, or LLM prompts. [DocLang](https://docs.abbyy.com/fine-parser/concepts/doclang) is the recommended format because it keeps that structure at a fraction of the token cost — a table encoded with DocLang's OTSL scheme needs five structural tokens where the equivalent HTML needs twenty-eight.

**A parser you run.** One container, CPU only, a small REST API, on your own machines or in your own cloud.

### Why teams use it

- Keeps tables, columns, and headings intact, and tells you where each one is on the page.
- Outputs [DocLang](https://docs.abbyy.com/fine-parser/concepts/doclang), a compact format built for LLM input and the one we recommend. JSON and plain text are also available.
- Runs as a single container on CPU hardware, on your own machines or in your own cloud.
- Two [recognition modes](https://docs.abbyy.com/fine-parser/basics/recognition-modes): fast mode for high volumes, high-fidelity mode when accuracy matters most.
- Recognizes more than 200 [languages](https://docs.abbyy.com/fine-parser/basics/languages), including Latin, Cyrillic, CJK, and Arabic scripts.
- Start on the free tier, upgrade to a paid plan as your volume grows, or move to [Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise) for high throughput and offline deployment.

---

## Quickstart

Get a license, run FineParser with Docker, and parse your first document into DocLang in under five minutes.

### Before you begin

- [Docker](https://docs.docker.com/get-docker/) installed and running
- A PDF or image file to parse
- A FineParser license file. Sign up for a plan at [fineparser.ai](https://www.fineparser.ai/#pricing) and you receive a `.fineparserlicense` file. See [Licenses and plans](https://docs.abbyy.com/fine-parser/basics/licenses-and-plans).
- Outbound access to ABBYY's license server. If your network restricts egress, allow the ranges listed under [Network requirements](https://docs.abbyy.com/fine-parser/reference/data-privacy#network-requirements) first.

### 1. Get a license

Sign up for a plan at [fineparser.ai](https://www.fineparser.ai/#pricing). The Free plan gives you 1,000 pages per month. You receive one `.fineparserlicense` file for your account.

### 2. Start FineParser

Pass the contents of your license file in the `FINEPARSER_LICENSE_DATA` environment variable, mount a volume for results, and publish port 8080:

```bash
docker run -d --name fineparser -p 8080:8080 \
  -e FINEPARSER_LICENSE_DATA="$(cat acme.fineparserlicense)" \
  -v fineparser-output:/app/output \
  abbyyteam/fineparser
```

> [!NOTE]
> On an ARM host, such as an Apple silicon Mac, add `--platform linux/amd64` to the command above. The image is amd64-only for now and runs under emulation, which is slower. See [Running on ARM](https://docs.abbyy.com/fine-parser/basics/configuration#running-on-arm).

Then confirm the license was accepted:

```bash
curl -s localhost:8080/healthz
# {"code":"ok","ready":true,"telemetry":true}
```

If `ready` is `false`, the response's `detail` field says why.

### 3. Submit a document

```bash
curl -s -X POST http://localhost:8080/parse \
  -F "file=@invoice.pdf" \
  -F "outputType=doclang"
```

FineParser accepts the document and returns a job ID right away:

```json
{"jobId":"6c4f1f0e-...","file":"invoice.pdf"}
```

### 4. Wait for it to finish

Poll the job with its ID until `status` is `successful`:

```bash
curl -s http://localhost:8080/jobs/6c4f1f0e-...
```

```json
{"status":"successful","file":"6c4f1f0e-.../invoice.pdf.doclang"}
```

A one-page invoice takes a few seconds. If `status` is `failed`, the response includes an `error` explaining why.

### 5. Download the result

Append the `file` value to `/jobs/`:

```bash
curl -OJ http://localhost:8080/jobs/6c4f1f0e-.../invoice.pdf.doclang
```

The file is the document's structure and text in DocLang:

```xml
<doclang>
  <heading level="1">Invoice 4471</heading>
  <table>
    <ched/>Item<ched/>Qty<ched/>Amount<nl/>
    <fcel/>Consulting<fcel/>12<fcel/>$4,800.00<nl/>
  </table>
</doclang>
```

Swap `doclang` for `json` or `txt` in the submit step to change the format.

---

## Two ways to run

### Server mode

Start the container without positional arguments and FineParser runs as an HTTP server on port 8080. It stays up until you stop it. Use this when your applications or other services need to send documents to FineParser over the network.

```bash
docker run -d --name fineparser -p 8080:8080 --stop-timeout 60 \
  -e FINEPARSER_LICENSE_DATA="$(cat acme.fineparserlicense)" \
  -v fineparser-output:/app/output \
  abbyyteam/fineparser
```

Recognition mode, languages, and the license are all set when the container starts, so requests carry no credentials or recognition settings. Results are written to `/app/output`, so mount a volume there or they disappear with the container.

**Parsing is asynchronous.** You submit a document and get a job ID back immediately, poll the job until it settles, then download the result. The HTTP connection is never held open for the length of a recognition.

| Step | Call | Result |
|---|---|---|
| Submit | `POST /parse` | `202 Accepted` with a job ID |
| Poll | `GET /jobs/{id}` | `status`: `pending`, `in-progress`, `successful`, `failed`, or `cancelled` |
| Download | `GET /jobs/{id}/{name}` | The result, streamed as an attachment |
| Delete | `DELETE /jobs/{id}` | Removes the job and its result from disk |
| Health | `GET /healthz` | Whether the container is licensed and ready |

Common `POST /parse` status codes:

| Status | Cause |
|---|---|
| `202` | Accepted. Poll `/jobs/{id}`. |
| `400` | `outputType` is missing or not one of `doclang`, `json`, or `txt`, or the request does not contain exactly one part named `file`. |
| `402` | Your plan has no pages left (`code: out_of_credits`). |
| `403` | The container has no license or it is not provisioned (`code: no_instance` or `not_provisioned`). |
| `413` | The request body is larger than the upload limit (1 GiB by default). |
| `503` | The license server cannot be reached or asked for a retry. Try again later. |

See [REST API](https://docs.abbyy.com/fine-parser/concepts/rest-api) for every endpoint, every status code, and Python and Node.js examples.

### CLI mode

Start the container with an input file and an output file as positional arguments and FineParser runs as a one-shot command instead. The container starts, parses the document, writes the result, and exits. Use this for scripts, batch jobs, and quick checks.

```bash
docker run --rm \
  -e FINEPARSER_LICENSE_DATA="$(cat acme.fineparserlicense)" \
  -v "$PWD:/data" \
  abbyyteam/fineparser /data/invoice.pdf /data/invoice.doclang
```

Both paths are inside the container, so mount the directory that holds your document and receives the output. The container runs as a non-root user, so the output directory must be writable by it. There is no `outputType` here — the format comes from the extension of the output file (`.doclang`, `.json`, `.txt`). Flags must come before the positional arguments.

Every run starts a new container and loads the recognition engine before it reads your document, so looping over a folder is slow. If you have more than a handful of documents, run the container as a server and post them to it instead.

See [CLI](https://docs.abbyy.com/fine-parser/concepts/cli).

---

## Output formats

Pass the `outputType` field with each submission. There is no default, so every request must name a format. Unlike recognition mode and languages, the format is chosen per request, so one container can serve all three. The downloaded file is named after the upload with the format appended, so `invoice.pdf` parsed to JSON arrives as `invoice.pdf.json`.

| Output type | Best for | Structure | Positions |
|---|---|---|---|
| `doclang` | LLM prompts, RAG, agents | Yes | Yes |
| `json` | Application code, pipelines | Yes | Yes |
| `txt` | Search indexing, plain extraction | No | No |

**DocLang** is an open, AI-native document standard from a working group founded by IBM, NVIDIA, Red Hat, ABBYY, HumanSignal, and Forgis, developed under Joint Development Foundation Projects as an LF AI & Data project. It is a constrained XML format built for LLM tokenizers: every element carries a semantic role, bounding box coordinates, and a position in the reading order, and tables keep their full grid structure. Because it is a specification rather than a product, code written against FineParser's output works with any other DocLang producer.

Learn more at [doclang.ai](https://doclang.ai/) and in the [doclang-project/doclang](https://github.com/doclang-project/doclang) repository. See [Output formats](https://docs.abbyy.com/fine-parser/basics/output-formats).

---

## Configuration

FineParser is configured once, when the container starts, with environment variables and command-line flags. Every setting has both forms and the flag wins when both are given. The settings hold for the life of the process.

| Setting | Flag | Environment variable | Default |
|---|---|---|---|
| License | `-license-data` | `FINEPARSER_LICENSE_DATA` | Required. The contents of your `.fineparserlicense` file. |
| Recognition mode | `-mode` | `FRE_MODE` | `accurate`. See [Recognition modes](https://docs.abbyy.com/fine-parser/basics/recognition-modes). |
| Languages | `-language` | `FRE_LANGUAGE` | `English`. See [Languages](https://docs.abbyy.com/fine-parser/basics/languages). |
| Listen address | `-addr` | `LISTEN_ADDR` | `:8080` |
| Output directory | `-output-dir` | `OUTPUT_DIR` | `/app/output`. Where results and the job database are written. Mount a volume here. |
| Scratch directory | `-scratch-dir` | `SCRATCH_DIR` | `/app/scratch`. Where uploads wait for recognition. Does not need a volume. |
| Job database | `-db` | `JOB_DB` | `<output-dir>/jobs.db` |
| Delete on download | `-delete-on-download` | `DELETE_ON_DOWNLOAD` | `false`. When `true`, a result and its job record are removed once fully downloaded. |
| Upload limit | `-max-upload-bytes` | `MAX_UPLOAD_BYTES` | `1Gi`. `0` is unlimited. |
| Upload timeout | `-upload-timeout` | `UPLOAD_TIMEOUT` | `5m`. `0` is unlimited. |
| Download stall timeout | `-download-stall-timeout` | `DOWNLOAD_STALL_TIMEOUT` | `5m`. `0` is unlimited. |

The license is multi-line text. Pass it with `$(cat acme.fineparserlicense)` or from a Kubernetes Secret. Docker `.env` files cannot carry multi-line values, so they do not work for this variable.

```bash
docker run -d --name fineparser -p 8080:8080 --stop-timeout 60 \
  -e FINEPARSER_LICENSE_DATA="$(cat acme.fineparserlicense)" \
  -v fineparser-output:/app/output \
  abbyyteam/fineparser \
  -mode fast -language "English,German"
```

**One configuration per container.** Recognition settings cannot be changed per request. Every document a container receives is processed with the mode and languages it was started with. If different document sets need different settings, run one container per configuration and route each request to the container that matches.

**Pick only the languages your documents actually contain.** Each added language widens the character set the engine considers on every page, which means slower processing, higher memory use, and lower accuracy.

**Mount a volume at `/app/output`.** It holds every result plus the job database. Without a named volume or host directory there, Docker creates an anonymous one and results are lost when the container is removed. Only one container can use a given output directory at a time — give each container its own volume.

**Give the container a 60-second stop timeout.** On a clean stop FineParser lets in-flight requests finish, flushes telemetry, and exits; Docker's 10-second default cuts that short. Use `--stop-timeout 60` with `docker run`, or `terminationGracePeriodSeconds: 60` in Kubernetes.

FineParser listens on port 8080 inside the container. Publish that port with `-p HOST:CONTAINER`. `-p 8080:8080` makes the API reachable on host port 8080 from any machine on the network; `-p 127.0.0.1:9000:8080` limits it to localhost on port 9000.

See [Configuration](https://docs.abbyy.com/fine-parser/basics/configuration).

---

## Requirements

- Docker, on CPU hardware. No GPU is required.
- The image is `linux/amd64`. On an ARM host (e.g. Apple silicon), add `--platform linux/amd64` — it runs under emulation, which is slower.
- Outbound access to ABBYY's license server:

  | Protocol | Address range |
  |---|---|
  | IPv4 | `185.146.155.0/24` |
  | IPv6 | `2620:122:f003::/48` |

> [!WARNING]
> If FineParser cannot reach these addresses, it does not parse any documents. Requests to `/parse` fail until connectivity is restored. For environments with no outbound connectivity, see [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise).

---

## Licenses and plans

FineParser is licensed by pages parsed per month. Your `.fineparserlicense` file gives the container access to your plan's monthly page allowance. Sign up at [fineparser.ai](https://www.fineparser.ai/#pricing).

| Plan | Pages per month | How to get it |
|---|---|---|
| Free | 1,000 | Sign up at fineparser.ai. Valid for one year. Credit card required. |
| Starter | 25,000 | Self-service upgrade |
| Growth | 250,000 | Self-service upgrade |
| Business | 1,000,000 | Self-service upgrade |
| Enterprise | Custom | [Talk to sales](https://docs.abbyy.com/fine-parser/reference/enterprise) |

Processing stops when you reach your plan's monthly page limit. FineParser does not offer pay-as-you-go pricing or overage billing — upgrade to a larger plan and the new allowance is available immediately.

- **Subscriptions, upgrades, downgrades, invoices, and payment methods** for self-service plans are managed through the [Stripe billing portal](https://billing.stripe.com/p/login/00weVd4rQeQC5zsf1y7EQ00). Sign in with the email address you registered with; Stripe emails a one-time login link.
- **Retrieving or managing a license file** — if you need to re-download a `.fineparserlicense` file or manage licenses at the account level — is done through the [ABBYY FlexNet Operations portal](https://abbyy.flexnetoperations.com/flexnet/operationsportal).
- Enterprise customers are invoiced directly; contact your account manager for changes.

See [Licenses and plans](https://docs.abbyy.com/fine-parser/basics/licenses-and-plans).

---

## Privacy

Documents are processed locally and never transmitted to ABBYY or anyone else. Two things do leave the container: license validation, which checks your remaining page balance and debits pages parsed, and [OpenTelemetry](https://opentelemetry.io/) usage telemetry emitted for every job. Telemetry is always on in release builds and cannot be disabled, but you can see exactly what is sent — print it to stdout with `FINEPARSER_TELEMETRY_STDOUT=on`, or copy it to your own collector with `FINEPARSER_OTLP_ENDPOINT`.

FineParser never collects document content, recognized text, filenames, file paths, file sizes, customer names, email addresses, account identifiers, user identities, or IP addresses. The license key is the one deliberate account-linked value.

See [Data privacy and telemetry](https://docs.abbyy.com/fine-parser/reference/data-privacy).

---

## FineParser vs. FineReader Engine

FineParser is built on [ABBYY FineReader Engine](https://docs.abbyy.com/fine-reader/engine/introduction), the OCR and document analysis SDK ABBYY has shipped to software vendors for more than two decades. FineParser packages a fixed, well-tested subset of the engine as a container and a REST API; most RAG and document-ingestion use cases need nothing more.

Consider the engine directly if you need to embed recognition in your own product, need export formats FineParser doesn't produce (searchable PDF/A, DOCX, XLSX, barcodes, ICR, and more), or need to tune the recognition pipeline beyond FineParser's two modes. See [FineReader Engine](https://docs.abbyy.com/fine-parser/reference/finereader-engine).

---

## Using AI coding agents

This repository ships an [`AGENTS.md`](./AGENTS.md) runbook. Point your coding agent at it and it knows how to deploy FineParser, guide you through getting a license, and run documents through the API. Claude Code, Codex, Cursor, and GitHub Copilot read it automatically when it is in your project root.

You can also connect the ABBYY documentation MCP server so your agent can look up FineParser endpoints, parameters, and output formats while it writes your integration. The server is read-only; it searches public documentation and does not touch your container, your documents, or your license.

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
| [Quickstart](https://docs.abbyy.com/fine-parser/getting-started/quickstart) | License, container, submit, poll, download, in five minutes. |
| [AI coding agents](https://docs.abbyy.com/fine-parser/getting-started/ai-agents) | Connect the docs MCP server to your agent. |
| **Basics** | |
| [Configuration](https://docs.abbyy.com/fine-parser/basics/configuration) | Every flag and environment variable, ports, ARM, shutdown. |
| [Recognition modes](https://docs.abbyy.com/fine-parser/basics/recognition-modes) | High fidelity vs. fast. |
| [Languages](https://docs.abbyy.com/fine-parser/basics/languages) | The full table of 200+ language keys. |
| [Output formats](https://docs.abbyy.com/fine-parser/basics/output-formats) | DocLang, JSON, and plain text compared. |
| [Licenses and plans](https://docs.abbyy.com/fine-parser/basics/licenses-and-plans) | Registration, allowances, subscriptions. |
| **Concepts** | |
| [DocLang](https://docs.abbyy.com/fine-parser/concepts/doclang) | The AI-native document format, annotated. |
| [REST API](https://docs.abbyy.com/fine-parser/concepts/rest-api) | Submit, poll, download, every status code, cURL/Python/Node.js. |
| [CLI](https://docs.abbyy.com/fine-parser/concepts/cli) | One-shot parsing from the command line. |
| **Reference** | |
| [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise) | Unlimited pages, air-gapped deployment. |
| [FineReader Engine](https://docs.abbyy.com/fine-parser/reference/finereader-engine) | When to use the SDK instead of the container. |
| [Data privacy and telemetry](https://docs.abbyy.com/fine-parser/reference/data-privacy) | What leaves the container, and what never does. |
| [Support and SLAs](https://docs.abbyy.com/fine-parser/reference/support) | Support by plan. |
| **Release notes** | |
| [1.0](https://docs.abbyy.com/fine-parser/release-notes/1.0) | Initial release. |

---

## Support

| Plan | Support |
|---|---|
| Free, Starter, Growth, Business | Community — [this repository](https://github.com/abbyy/FineParser/issues) |
| Enterprise | Enterprise SLAs and ABBYY technical support |

See [SUPPORT.md](./SUPPORT.md) for what to include in a report, and [CONTRIBUTING.md](./CONTRIBUTING.md) for how to help.

---

## License

The contents of this repository are licensed under [Apache-2.0](./LICENSE). The FineParser Docker image is a commercial ABBYY product, licensed separately under the terms you accept when you sign up for a plan.

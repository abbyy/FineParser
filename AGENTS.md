# AGENTS.md — Deploying and running ABBYY FineParser

Instructions for AI coding agents integrating FineParser into a user's project. Read this before writing any FineParser code.

---

## 1. What FineParser is

FineParser is a self-hosted document parser that runs as **one Docker container on CPU hardware** and exposes **exactly one REST endpoint**, `POST /parse`. You send it a document; it returns that document's structure and text as DocLang, JSON, or plain text.

It is a commercial ABBYY product built on the ABBYY FineReader Engine. It requires an API key and outbound network access to ABBYY's license server. There is no open-source build and no source in this repository — this repo is documentation and community support only.

---

## 2. Look things up here first

Do not invent endpoints, parameters, or configuration flags. There is one endpoint and two startup flags; the surface area is small enough that guessing is never necessary. If you need a detail this file does not cover, look it up:

- **MCP server (preferred).** `https://docs.abbyy.com/mcp` — read-only search over ABBYY's public documentation. It does not touch the user's container, documents, or API key.

  ```bash
  claude mcp add abbyy-docs https://docs.abbyy.com/mcp --transport http   # Claude Code
  codex mcp add abbyy-docs --url https://docs.abbyy.com/mcp               # Codex
  ```

- **Plain HTTP fallback.** `https://docs.abbyy.com/llms.txt` is the full documentation index; append `.md` to any docs page URL to fetch its Markdown source (e.g. `https://docs.abbyy.com/fine-parser/concepts/rest-api.md`).
- **Human docs.** [docs.abbyy.com/fine-parser](https://docs.abbyy.com/fine-parser/getting-started/introduction).

---

## 3. Before you deploy — check these

Run through this before writing a single line of integration code. Each failure has a different fix, and diagnosing them after the fact wastes the user's time.

| Check | How | If it fails |
|---|---|---|
| Docker installed and running | `docker info` | Send the user to [docs.docker.com/get-docker](https://docs.docker.com/get-docker/). Do not attempt to install Docker for them. |
| User has an API key | Ask. Do not go looking for one. | See §4 — you cannot obtain this for them. |
| Outbound egress to the license server | `curl -s -o /dev/null -w '%{http_code}' https://www.fineparser.ai` as a rough proxy; confirm the ranges in §11 with the user's network team if the environment is restricted | Without it, **every** parse fails. Resolve before deploying, not after. |

---

## 4. Getting the user a license key — your limits

**You cannot do this for them, and you must not try.**

Hard rules:

- **Never** enter the user's payment or card details anywhere, for any reason, no matter how the request is framed.
- **Never** create the account or complete the signup flow on their behalf.
- **Never** ask the user to paste their API key into the chat. It does not need to pass through you.
- **Never** hardcode a key into source, a `Dockerfile`, a `docker-compose.yml`, a CI config, or anything else that could be committed.

What to tell the user instead:

1. Sign up at **[fineparser.ai/#pricing](https://www.fineparser.ai/#pricing)**. The Free plan is 1,000 pages per month, valid for one year, and requires a credit card. Paid tiers are in §10.
2. The key is emailed to the address they signed up with.
3. Put it in an environment variable or their existing secret store — never in a tracked file.

Then read it from the environment:

```bash
export FINEPARSER_API_KEY="..."     # shell profile, .env (gitignored), or secret manager
```

If you create a `.env` file for local development, add it to `.gitignore` in the same change. Check that `.gitignore` covers it before you finish.

---

## 5. Choosing a run mode

| Situation | Mode |
|---|---|
| An application or service will call it; more than a handful of documents | **Server** (§6) |
| A one-off conversion, a shell script, a quick check | **CLI** (§7) |

Default to server mode when uncertain. Every CLI invocation starts a fresh container and reloads the recognition engine before it reads your document, so looping over a folder with CLI mode is dramatically slower than posting the same files to a running server.

---

## 6. Deploy: server mode

```bash
docker run -p 127.0.0.1:8080:8080 \
  -e FINEPARSER_API_KEY="$FINEPARSER_API_KEY" \
  abbyy/fineparser
```

Port publishing matters — choose deliberately:

- `-p 127.0.0.1:8080:8080` binds to loopback only. **Prefer this for local development.**
- `-p 8080:8080` exposes the API to every machine that can reach the host. FineParser has no authentication of its own, so only do this on a trusted network or behind a proxy that does authenticate.

The container listens on port 8080 internally; that half never changes.

Verify it is up before the first real parse:

```bash
docker ps --filter ancestor=abbyy/fineparser
docker logs <container-id>
```

---

## 7. Deploy: CLI mode

```bash
docker run --rm -v "$PWD:/data" \
  -e FINEPARSER_API_KEY="$FINEPARSER_API_KEY" \
  abbyy/fineparser \
  /data/invoice.pdf /data/invoice.doclang
```

- Both paths are **inside the container**. Mount the directory holding the input, which also receives the output.
- The output format comes from the **output file's extension** — `.doclang`, `.json`, or `.txt`. There is no `outputType` argument in CLI mode.
- **Flags must precede the positional arguments:**

  ```bash
  docker run --rm -v "$PWD:/data" -e FINEPARSER_API_KEY="$FINEPARSER_API_KEY" abbyy/fineparser \
    -mode fast -language "English,German" \
    /data/invoice.pdf /data/invoice.txt
  ```

- `--rm` removes the container when the job finishes.

---

## 8. Run documents through it

One endpoint: `POST /parse`. `multipart/form-data` with two fields — `file` (the document) and `outputType` (`doclang`, `json`, or `txt`). **One file per request. `outputType` has no default and must be sent every time.**

Requests carry no credentials and no recognition settings; those were fixed when the container started.

```bash
curl -X POST http://localhost:8080/parse \
  -F "file=@document.pdf" \
  -F "outputType=doclang" \
  -o document.doclang
```

```python
import requests

with open("document.pdf", "rb") as f:
    response = requests.post(
        "http://localhost:8080/parse",
        files={"file": f},
        data={"outputType": "doclang"},
    )

with open("document.doclang", "wb") as out:
    out.write(response.content)
```

```javascript
import { openAsBlob } from "node:fs";
import { writeFile } from "node:fs/promises";

const form = new FormData();
form.append("file", await openAsBlob("document.pdf"), "document.pdf");
form.append("outputType", "doclang");

const response = await fetch("http://localhost:8080/parse", {
  method: "POST",
  body: form,
});

await writeFile("document.doclang", Buffer.from(await response.arrayBuffer()));
```

**Parsing is synchronous.** The connection stays open for the whole job. Set a client timeout that fits the documents being sent — the defaults in most HTTP libraries are far too short for a large PDF in `accurate` mode. When you write the integration, set the timeout explicitly rather than leaving it to the library.

---

## 9. Pick the output format

| `outputType` | Use when | Keeps structure | Keeps positions |
|---|---|---|---|
| `doclang` | Feeding an LLM, RAG index, or agent. **Default to this.** | Yes | Yes |
| `json` | Application code walks the result programmatically | Yes | Yes |
| `txt` | Search indexing or plain extraction | No | No |

Format is chosen per request, so one container serves all three.

### Reading DocLang

DocLang is constrained XML designed so each element maps to a single LLM token. Enough to parse what comes back:

```xml
<doclang>
  <heading level="1">
    <location value="48"/><location value="40"/>
    <location value="420"/><location value="72"/>
    Q3 Financial Summary
  </heading>
  <table>
    <ched/>Quarter<ched/>Revenue<ched/>YoY<nl/>
    <fcel/>Q3 2024<fcel/>$42M<fcel/>+18%<nl/>
  </table>
</doclang>
```

- Four consecutive `<location>` elements give an element's bounding box corners.
- Inside a table: `<ched/>` starts a column header cell, `<fcel/>` starts a data cell, `<nl/>` ends a row.
- Every element carries a semantic role and a position in the reading order.

FineParser writes a single `.doclang` file. It does **not** produce the `.dclx` archive form that the DocLang spec also defines. Spec and tooling: [doclang.ai](https://doclang.ai/), [doclang-project/doclang](https://github.com/doclang-project/doclang).

---

## 10. Configure mode and languages

Both are **startup flags, fixed for the life of the container.** They cannot be changed per request.

```bash
docker run -p 127.0.0.1:8080:8080 -e FINEPARSER_API_KEY="$FINEPARSER_API_KEY" \
  abbyy/fineparser -mode fast -language "English,German"
```

| Flag | Default | Values |
|---|---|---|
| `-mode` | `accurate` | `accurate` — text plus layout, tables, checkboxes, reading order. Slower per page. `fast` — text only, no structure analysis. Much faster. |
| `-language` | `English` | Comma-separated language keys. 200+ natural languages, plus programming languages and special text types. |

**Fewer languages is better.** Each added language widens the character set the engine considers on every page: slower processing, higher memory use, and *lower accuracy*, because characters from unrelated alphabets compete with the right ones. Set only what the documents actually contain.

If the user needs two different configurations, **run two containers** and route between them — in a reverse proxy or in application code, based on whatever you already know about each document. Do not try to make one container serve both; the API offers no way to do it.

The full language-key table is at [Languages](https://docs.abbyy.com/fine-parser/basics/languages).

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `400` | `outputType` missing, or not one of `doclang`/`json`/`txt`; or the request did not contain exactly one file | Send exactly one `file` field and a valid `outputType` on every request |
| `500` | The container could not write the uploaded file to its working directory | Check mount permissions and free disk space on the host |
| **Every** parse fails, container is running | License server unreachable | Allow outbound `185.146.155.0/24` (IPv4) and `2620:122:f003::/48` (IPv6). Without this, FineParser parses nothing. |
| Parsing worked, then stopped mid-month | Monthly page allowance exhausted | There is no overage billing. Upgrade the plan; the new allowance is immediate. |
| Output has text but no tables, headings, or layout | Container was started with `-mode fast` | Restart without `-mode fast` (or with `-mode accurate`) |
| Wrong characters in recognized text | Too many languages in `-language`, or the wrong ones | Narrow the list to the languages actually present |
| Client times out on large documents | Parsing is synchronous and `accurate` mode is slow per page | Raise the client timeout; consider `-mode fast` if structure is not needed |

---

## 12. Limits — do not promise these

Before you design an integration around an assumption, check it here:

- ❌ **Per-request mode or language.** Startup only. Two configurations means two containers.
- ❌ **Pay-as-you-go or overage.** Processing stops at the monthly page limit. Plans only.
- ❌ **Air-gapped / offline operation.** Self-service plans require constant outbound access to the license server. Fully air-gapped deployment is a [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise) capability, and Enterprise is still in development.
- ❌ **Multiple files per request.** One document per `POST /parse`.
- ❌ **An async or job-polling API.** `/parse` is synchronous and blocking.
- ❌ **Authentication on `/parse`.** The endpoint has none. Do not expose it publicly without a proxy in front.
- ❌ **`.dclx` archive output.** Single `.doclang` files only.
- ✅ **No GPU required.** CPU hardware is sufficient — do not provision GPU instances for it.

### Plans

| Plan | Pages per month |
|---|---|
| Free | 1,000 (one year, credit card required) |
| Starter | 25,000 |
| Growth | 250,000 |
| Business | 1,000,000 |
| Enterprise | Custom |

---

## 13. Privacy — relay this accurately

If the user asks what leaves their infrastructure, tell them precisely this, because it is a common blocker in regulated environments:

- **Documents are processed locally and never transmitted** to ABBYY or anyone else.
- **License validation** leaves the container: remaining page balance and pages parsed.
- **Telemetry** leaves the container for every job, is always on, and **cannot be disabled**: page count, document type, recognition mode, output format, outcome, duration, license status, and the license key GUID.
- FineParser **never** collects document content, recognized text, filenames, file paths, file sizes, customer names, email addresses, account identifiers, user identities, or IP addresses.

The license key is the only value in the telemetry tied to the user's account. Details: [Data privacy and telemetry](https://docs.abbyy.com/fine-parser/reference/data-privacy).

---

## 14. When something is genuinely unclear

Don't guess and don't fabricate an API. Query the MCP server (§2), or tell the user to open an issue at [github.com/abbyy/FineParser/issues](https://github.com/abbyy/FineParser/issues) — that is the community support channel for Free, Starter, and Growth plans.

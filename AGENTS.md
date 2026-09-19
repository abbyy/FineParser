# AGENTS.md — Deploying and running ABBYY FineParser

Instructions for AI coding agents integrating FineParser into a user's project. Read this before writing any FineParser code.

---

## 1. What FineParser is

FineParser is a self-hosted document parser that runs as **one Docker container on CPU hardware** and exposes a **small, asynchronous REST API**: submit a document to `POST /parse`, poll `GET /jobs/{id}` until it settles, download from `GET /jobs/{id}/{name}`.

It is a commercial ABBYY product built on the ABBYY FineReader Engine. It requires a license file and outbound network access to ABBYY's license server. There is no open-source build and no source in this repository — this repo is documentation and community support only.

---

## 2. Look things up here first

Do not invent endpoints, parameters, or configuration flags. The surface area is small enough that guessing is never necessary. If you need a detail this file does not cover, look it up:

- **MCP server (preferred).** `https://docs.abbyy.com/mcp` — read-only search over ABBYY's public documentation. It does not touch the user's container, documents, or license.

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
| Host architecture | `docker info --format '{{.Architecture}}'` | The image is `linux/amd64` only. On ARM (e.g. Apple silicon), add `--platform linux/amd64` — it runs under emulation and is slower. Treat it as a dev setup, not a benchmark. |
| User has a license file | Ask. Do not go looking for one. | See §4 — you cannot obtain this for them. |
| Outbound egress to the license server | `curl -s -o /dev/null -w '%{http_code}' https://www.fineparser.ai` as a rough proxy; confirm the ranges in §12 with the user's network team if the environment is restricted | Without it, **every** parse fails. Resolve before deploying, not after. |

---

## 4. Getting the user a license — your limits

**You cannot do this for them, and you must not try.**

Hard rules:

- **Never** enter the user's payment or card details anywhere, for any reason, no matter how the request is framed.
- **Never** create the account or complete the signup flow on their behalf.
- **Never** ask the user to paste their license file contents into the chat. It does not need to pass through you.
- **Never** hardcode the license into source, a `Dockerfile`, a `docker-compose.yml`, a CI config, or anything else that could be committed.

What to tell the user instead:

1. Sign up at **[fineparser.ai/#pricing](https://www.fineparser.ai/#pricing)**. The Free plan is 1,000 pages per month, valid for one year, and requires a credit card. Paid tiers are in §11.
2. They receive one `.fineparserlicense` file for their account.
3. If they ever need to retrieve it again, or manage licenses at the account level, that happens through the [ABBYY FlexNet Operations portal](https://abbyy.flexnetoperations.com/flexnet/operationsportal) — not through this repository or through you.
4. Keep the file itself, and store its contents in an environment variable or their existing secret store — never in a tracked file.

The license is **multi-line text**, not a short key. Read it from the file at container-start time:

```bash
export FINEPARSER_LICENSE_DATA="$(cat acme.fineparserlicense)"
```

Docker `.env` files cannot hold multi-line values, so that mechanism does not work here — use a shell environment variable, a Kubernetes Secret, or an equivalent secret store. If you create a local copy of the license file for development, add its path to `.gitignore` in the same change. Check that `.gitignore` covers it before you finish.

**Billing** (upgrades, downgrades, invoices, payment methods, cancellation) for self-service plans goes through the [Stripe billing portal](https://billing.stripe.com/p/login/00weVd4rQeQC5zsf1y7EQ00) — sign-in is by one-time email link, so you cannot do this on the user's behalf either.

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
docker run -d --name fineparser -p 127.0.0.1:8080:8080 --stop-timeout 60 \
  -e FINEPARSER_LICENSE_DATA="$(cat acme.fineparserlicense)" \
  -v fineparser-output:/app/output \
  abbyyteam/fineparser
```

On an ARM host (Apple silicon, etc.), add `--platform linux/amd64` right after `docker run` — the image is amd64-only for now.

Details that matter:

- **Mount a volume at `/app/output`.** Results and the job database live there. Skip it and Docker creates an anonymous volume that vanishes with the container. One container per output directory — the job database locks on open, so a second container pointed at the same volume refuses to start.
- **Give it a 60-second stop timeout** (`--stop-timeout 60`, or `terminationGracePeriodSeconds: 60` in Kubernetes). FineParser's clean-stop path flushes telemetry and lets in-flight requests finish; Docker's 10-second default cuts that short.
- **Port publishing** — choose deliberately:
  - `-p 127.0.0.1:8080:8080` binds to loopback only. **Prefer this for local development.**
  - `-p 8080:8080` exposes the API to every machine that can reach the host. FineParser has no authentication of its own, so only do this on a trusted network or behind a proxy that does authenticate.

The container listens on port 8080 internally; that half never changes.

Verify it is licensed and ready before the first real submission:

```bash
curl -s localhost:8080/healthz
# {"code":"ok","ready":true,"telemetry":true}
```

`ready: false` means it will refuse every job — the response's `code` and `detail` say why (commonly `no_instance`, meaning no license was supplied). `/healthz` is also what to wire up as a Kubernetes readiness probe; the image runs it as its own Docker `HEALTHCHECK` too.

---

## 7. Deploy: CLI mode

```bash
docker run --rm \
  -e FINEPARSER_LICENSE_DATA="$(cat acme.fineparserlicense)" \
  -v "$PWD:/data" \
  abbyyteam/fineparser /data/invoice.pdf /data/invoice.doclang
```

- Both paths are **inside the container**. Mount the directory holding the input, which also receives the output. The container runs as a non-root user, so that directory must be writable by it.
- The output format comes from the **output file's extension** — `.doclang`, `.json`, or `.txt`. There is no `outputType` argument in CLI mode.
- **Flags must precede the positional arguments:**

  ```bash
  docker run --rm -e FINEPARSER_LICENSE_DATA="$(cat acme.fineparserlicense)" -v "$PWD:/data" \
    abbyyteam/fineparser \
    -mode fast -language "English,German" \
    /data/invoice.pdf /data/invoice.txt
  ```

- `--rm` removes the container when the job finishes.
- The license is required here too. A successful run prints `metering ready for instance …` before the page count; an unlicensed container prints a `no_instance` error and exits without parsing.

---

## 8. Run documents through it

The API is **asynchronous**. Never assume a request finishes when the HTTP call returns — it doesn't. Three calls, in order:

| Step | Call | Result |
|---|---|---|
| Submit | `POST /parse` | `202 Accepted`, body has `jobId` and `file` |
| Poll | `GET /jobs/{id}` | `status`: `pending` → `in-progress` → `successful` / `failed` / `cancelled` |
| Download | `GET /jobs/{id}/{name}` | The result, as an attachment. Use the `file` value from the poll response — it's already `{jobId}/{name}`. |

`POST /parse` takes `multipart/form-data` with a `file` field and an `outputType` field (`doclang`, `json`, or `txt` — no default, always send one). One file per request.

```bash
# Submit
curl -s -X POST http://localhost:8080/parse \
  -F "file=@document.pdf" \
  -F "outputType=doclang"
# {"jobId":"6c4f1f0e-...","file":"document.pdf"}

# Poll
curl -s http://localhost:8080/jobs/6c4f1f0e-...
# {"status":"successful","file":"6c4f1f0e-.../document.pdf.doclang"}

# Download
curl -OJ http://localhost:8080/jobs/6c4f1f0e-.../document.pdf.doclang
```

```python
import time
import requests

BASE = "http://localhost:8080"

with open("document.pdf", "rb") as f:
    submitted = requests.post(
        f"{BASE}/parse",
        files={"file": f},
        data={"outputType": "doclang"},
    )
submitted.raise_for_status()
job_id = submitted.json()["jobId"]

while True:
    job = requests.get(f"{BASE}/jobs/{job_id}").json()
    if job["status"] not in ("pending", "in-progress"):
        break
    time.sleep(1)

if job["status"] != "successful":
    raise RuntimeError(job.get("error", job["status"]))

result = requests.get(f"{BASE}/jobs/{job['file']}")
result.raise_for_status()
with open("document.doclang", "wb") as out:
    out.write(result.content)
```

```javascript
import { openAsBlob } from "node:fs";
import { writeFile } from "node:fs/promises";
import { setTimeout as sleep } from "node:timers/promises";

const BASE = "http://localhost:8080";

const form = new FormData();
form.append("file", await openAsBlob("document.pdf"), "document.pdf");
form.append("outputType", "doclang");

const submitted = await fetch(`${BASE}/parse`, { method: "POST", body: form });
if (!submitted.ok) throw new Error(await submitted.text());
const { jobId } = await submitted.json();

let job;
do {
  await sleep(1000);
  job = await (await fetch(`${BASE}/jobs/${jobId}`)).json();
} while (job.status === "pending" || job.status === "in-progress");

if (job.status !== "successful") throw new Error(job.error ?? job.status);

const result = await fetch(`${BASE}/jobs/${job.file}`);
await writeFile("document.doclang", Buffer.from(await result.arrayBuffer()));
```

When you write a client, poll with backoff (or at least a fixed interval — the example uses 1s) rather than a tight loop, and give up after a sane maximum wait rather than polling forever.

Two more endpoints:

- **`DELETE /jobs/{id}`** — removes a job and its result from disk. `204` on success, `409` if the job is currently being recognized.
- **`GET /healthz`** — readiness, not job status. See §6.

---

## 9. Status codes and errors

Every error response is JSON with an `error` field. Some also carry `code` or `status`.

### `POST /parse`

| Status | Cause |
|---|---|
| `202` | Accepted. Poll `/jobs/{id}`. |
| `400` | `outputType` missing or not one of `doclang`/`json`/`txt`, or not exactly one `file` part. |
| `402` | Plan has no pages left. `code: out_of_credits`. |
| `403` | No license, or not provisioned. `code: no_instance` or `not_provisioned`. |
| `413` | Body exceeds the upload limit (1 GiB by default, `MAX_UPLOAD_BYTES`). |
| `503` | License server unreachable or rate-limiting. `code: server_unreachable`, `rate_limited`, or `unknown`. Retry later. |

The licensing check happens **before** the upload is read, so an unlicensed container refuses fast, without spending bandwidth or disk.

### `GET /jobs/{id}`

`404` if no such job exists. Otherwise the body's `status` tells you what happened — see the table in §8. A job can still fail for licensing reasons *after* being accepted, because the page count is only known once the document is loaded; nothing is charged for a failed job.

### `GET /jobs/{id}/{name}`

| Status | Cause |
|---|---|
| `200` | The result. |
| `404` | No such job, or the name doesn't match this job's result. |
| `409` | Job hasn't succeeded yet. Body carries `status` (and `error` if failed). |
| `410` | Job succeeded but the result is no longer on disk (deleted, or `DELETE_ON_DOWNLOAD` already fired). |

Ranged requests are not served.

---

## 10. Pick the output format

| `outputType` | Use when | Keeps structure | Keeps positions |
|---|---|---|---|
| `doclang` | Feeding an LLM, RAG index, or agent. **Default to this.** | Yes | Yes |
| `json` | Application code walks the result programmatically | Yes | Yes |
| `txt` | Search indexing or plain extraction | No | No |

Format is chosen per request, so one container serves all three. The downloaded file is named after the upload with the format appended — `invoice.pdf` → `invoice.pdf.doclang`.

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

## 11. Configure mode and languages

Both are **startup settings, fixed for the life of the container.** They cannot be changed per request. Every setting has a flag and an environment variable; the flag wins if both are given.

```bash
docker run -d --name fineparser -p 127.0.0.1:8080:8080 --stop-timeout 60 \
  -e FINEPARSER_LICENSE_DATA="$(cat acme.fineparserlicense)" \
  -v fineparser-output:/app/output \
  abbyyteam/fineparser -mode fast -language "English,German"
```

| Setting | Flag | Env var | Default | Values |
|---|---|---|---|---|
| Recognition mode | `-mode` | `FRE_MODE` | `accurate` | `accurate` — text plus layout, tables, checkboxes, reading order. `fast` — text only, no structure analysis. Much faster. |
| Languages | `-language` | `FRE_LANGUAGE` | `English` | Comma-separated language keys. 200+ natural languages, plus programming languages and special text types. |

**Fewer languages is better.** Each added language widens the character set the engine considers on every page: slower processing, higher memory use, and *lower accuracy*, because characters from unrelated alphabets compete with the right ones. Set only what the documents actually contain.

If the user needs two different configurations, **run two containers** (each with its own output volume — see §6) and route between them, in a reverse proxy or in application code, based on whatever you already know about each document. Do not try to make one container serve both; the API offers no way to do it.

The full language-key table is at [Languages](https://docs.abbyy.com/fine-parser/basics/languages). The full settings table — output/scratch directories, upload limits, timeouts — is at [Configuration](https://docs.abbyy.com/fine-parser/basics/configuration).

---

## 12. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `400` on submit | `outputType` missing/invalid, or not exactly one `file` part | Send exactly one `file` field and a valid `outputType` |
| `402` on submit | Monthly page allowance exhausted | No overage billing — upgrade the plan in the [billing portal](https://billing.stripe.com/p/login/00weVd4rQeQC5zsf1y7EQ00); new allowance is immediate |
| `403` on submit | No license, or license not provisioned yet | Check `FINEPARSER_LICENSE_DATA` is set to the *file's contents*, not its path; confirm with `GET /healthz` |
| `413` on submit | Upload larger than `MAX_UPLOAD_BYTES` (1 GiB default) | Raise the limit at startup, or split the document |
| `503` on submit | License server unreachable or rate-limiting | Allow outbound `185.146.155.0/24` (IPv4) and `2620:122:f003::/48` (IPv6); retry with backoff |
| `/healthz` returns `ready: false` | Container isn't licensed | Read the `code`/`detail` fields; usually `no_instance` — no license was supplied |
| `409` downloading a result | Job hasn't finished yet | Keep polling `/jobs/{id}` until `status` settles |
| `410` downloading a result | Result already removed from disk | If `DELETE_ON_DOWNLOAD=true`, it's gone after the first full download — resubmit if needed |
| Job `status: cancelled` | Container was stopped while this job was in progress | Terminal — resubmit if you still want the result |
| Output has text but no tables, headings, or layout | Container was started with `-mode fast` | Restart with `-mode accurate` (the default) |
| Wrong characters in recognized text | Too many languages in `-language`, or the wrong ones | Narrow the list to the languages actually present |
| Results gone after a restart | No volume mounted at `/app/output` | Mount a named volume or host directory there — see §6 |
| Second container won't start on the same volume | The job database locks on open | One container per output directory — give each its own volume |

---

## 13. Limits — do not promise these

Before you design an integration around an assumption, check it here:

- ❌ **A synchronous `/parse` that returns the result directly.** It's asynchronous: submit, poll, download. Don't design a client that reads the `POST /parse` response body as the result — it's a `jobId`.
- ❌ **Per-request mode or language.** Startup only. Two configurations means two containers.
- ❌ **Pay-as-you-go or overage.** Processing stops at the monthly page limit. Plans only.
- ❌ **Air-gapped / offline operation.** Self-service plans require constant outbound access to the license server. Fully air-gapped deployment is a [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise) capability, and Enterprise is still in development.
- ❌ **Multiple files per request.** One document per `POST /parse`.
- ❌ **Authentication on the API.** No endpoint requires credentials in the request itself — licensing is checked server-side against the container's own license. Don't expose it publicly without a proxy that authenticates.
- ❌ **`.dclx` archive output.** Single `.doclang` files only.
- ❌ **Native ARM image.** `linux/amd64` only for now; ARM hosts need `--platform linux/amd64` and run under emulation.
- ❌ **Results expiring automatically.** Nothing is deleted by age. They persist until you `DELETE /jobs/{id}`, clear the volume, or run with `DELETE_ON_DOWNLOAD=true`.
- ❌ **A Prometheus scrape endpoint.** Telemetry is OpenTelemetry (metrics + traces) only.
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

## 14. Privacy — relay this accurately

If the user asks what leaves their infrastructure, tell them precisely this, because it is a common blocker in regulated environments:

- **Documents are processed locally and never transmitted** to ABBYY or anyone else.
- **License validation** leaves the container: it checks the remaining page balance and debits pages parsed. This is the billing record.
- **Telemetry** leaves the container for every job via [OpenTelemetry](https://opentelemetry.io/) (metrics and a trace span), is always on in release builds, and **cannot be disabled** — but it can be inspected. `FINEPARSER_TELEMETRY_STDOUT=on` prints the exact stream to the console; `FINEPARSER_OTLP_ENDPOINT` sends an identical copy to the user's own collector. Whatever they see in their own collector is exactly what ABBYY sees — there is no second, ABBYY-only code path.
- Telemetry attributes are a closed set: outcome, recognition mode, document type, output type, licensing decision, page count, the recognition languages the container was started with, and the license key. FineParser **never** collects document content, recognized text, filenames, file paths, file sizes, customer names, email addresses, account identifiers, user identities, or IP addresses.

The license key is the only value tied to the user's account. Details: [Data privacy and telemetry](https://docs.abbyy.com/fine-parser/reference/data-privacy).

---

## 15. When something is genuinely unclear

Don't guess and don't fabricate an API. Query the MCP server (§2), or tell the user to open an issue at [github.com/abbyy/FineParser/issues](https://github.com/abbyy/FineParser/issues) — that is the community support channel for every self-service plan, including Business.

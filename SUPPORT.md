# Support

Where to get help with ABBYY FineParser, and what to include so we can actually help.

## Check the documentation first

Most questions are answered at **[docs.abbyy.com/fine-parser](https://docs.abbyy.com/fine-parser/getting-started/introduction)**. The pages people reach for most often:

- [Quickstart](https://docs.abbyy.com/fine-parser/getting-started/quickstart) — license, container, submit, poll, download
- [REST API](https://docs.abbyy.com/fine-parser/concepts/rest-api) — submit, poll, download, every status code
- [Configuration](https://docs.abbyy.com/fine-parser/basics/configuration) — every setting, ports, ARM, shutdown
- [Output formats](https://docs.abbyy.com/fine-parser/basics/output-formats) — DocLang, JSON, plain text
- [Licenses and plans](https://docs.abbyy.com/fine-parser/basics/licenses-and-plans) — allowances and billing
- [Data privacy and telemetry](https://docs.abbyy.com/fine-parser/reference/data-privacy) — network requirements

## Support by plan

| Plan | Support |
|---|---|
| Free, Starter, Growth, Business | Community — [this repository](https://github.com/abbyy/FineParser/issues) |
| Enterprise | Enterprise SLAs, named contacts, and ABBYY technical support |

### Community support

[Issues in this repository](https://github.com/abbyy/FineParser/issues) are where to ask questions, report bugs, request features, and see what other people have run into. It is monitored by ABBYY, but response times are best-effort rather than contractual. This is the support channel for every self-service plan, including Business.

**Search existing issues before opening a new one.** Several of the most common problems — every parse failing, output missing its structure, processing stopping mid-month — already have answers.

### Enterprise SLAs

Enterprise customers get contractual response and resolution targets, escalation paths, and named contacts. See [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise).

## What to include in a report

The more of this you provide, the faster this gets resolved:

- **FineParser version / image tag** (e.g. `abbyyteam/fineparser:1.0.0`)
- **How you ran it** — server mode or CLI mode, and the exact `docker run` command, with your license contents redacted
- **Recognition settings** — your `-mode` and `-language` values
- **The `outputType`** you requested
- **For a job that failed or got stuck** — the `jobId`, the `status` from `GET /jobs/{id}`, and the response from `GET /healthz`
- **Host OS, architecture, and Docker version** (`docker version`) — note if you're on ARM and using `--platform linux/amd64`
- **What you expected, and what happened instead** — including the exact error text or status code
- **Container logs** (`docker logs <container-name>`), redacted
- **A sample document**, if you are able to share one

> [!CAUTION]
> Never include your license file contents, license key GUID, or confidential documents in a public issue. Redact them first. If reproducing a problem genuinely requires a sensitive document, say so in the issue and we will arrange a private channel.

## Billing and license questions

- **Subscriptions, upgrades, downgrades, invoices, and payment methods** for self-service plans are handled through the [Stripe billing portal](https://billing.stripe.com/p/login/00weVd4rQeQC5zsf1y7EQ00) — not through this repository.
- **Retrieving or managing a `.fineparserlicense` file** is handled through the [ABBYY FlexNet Operations portal](https://abbyy.flexnetoperations.com/flexnet/operationsportal).
- Enterprise customers are invoiced directly; contact your account manager for changes.

See [Licenses and plans](https://docs.abbyy.com/fine-parser/basics/licenses-and-plans).

## Interested in Enterprise?

Unlimited pages, orchestrated throughput of 500+ pages per second, fully air-gapped deployment, and engine tuning. It is in development now, and early conversations shape what ships — [contact us](https://www.abbyy.com/company/contact-us/) and tell us about your use case. See [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise).

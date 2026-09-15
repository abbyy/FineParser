# Support

Where to get help with ABBYY FineParser, and what to include so we can actually help.

## Check the documentation first

Most questions are answered at **[docs.abbyy.com/fine-parser](https://docs.abbyy.com/fine-parser/getting-started/introduction)**. The pages people reach for most often:

- [Quickstart](https://docs.abbyy.com/fine-parser/getting-started/quickstart) — key, container, first parse
- [REST API](https://docs.abbyy.com/fine-parser/concepts/rest-api) — `POST /parse`, parameters, status codes
- [Configuration](https://docs.abbyy.com/fine-parser/basics/configuration) — startup flags and ports
- [Output formats](https://docs.abbyy.com/fine-parser/basics/output-formats) — DocLang, JSON, plain text
- [API keys and plans](https://docs.abbyy.com/fine-parser/basics/api-keys-and-plans) — allowances and billing
- [Data privacy and telemetry](https://docs.abbyy.com/fine-parser/reference/data-privacy) — network requirements

## Support by plan

| Plan | Support |
|---|---|
| Free, Starter, Growth | Community — [this repository](https://github.com/abbyy/FineParser/issues) |
| Business | Community and [ABBYY technical support](https://support.abbyy.com/hc/en-us) |
| Enterprise | Enterprise SLAs, named contacts, and ABBYY technical support |

### Community support

[Issues in this repository](https://github.com/abbyy/FineParser/issues) are where to ask questions, report bugs, request features, and see what other people have run into. It is monitored by ABBYY, but response times are best-effort rather than contractual.

**Search existing issues before opening a new one.** Several of the most common problems — every parse failing, output missing its structure, processing stopping mid-month — already have answers.

### ABBYY technical support

Business plan customers can open a ticket with [ABBYY technical support](https://support.abbyy.com/hc/en-us) in addition to using the community channel.

### Enterprise SLAs

Enterprise customers get contractual response and resolution targets, escalation paths, and named contacts. See [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise).

## What to include in a report

The more of this you provide, the faster this gets resolved:

- **FineParser version / image tag** (e.g. `abbyy/fineparser:1.0.0`)
- **How you ran it** — server mode or CLI mode, and the exact `docker run` command, with your API key redacted
- **Recognition settings** — your `-mode` and `-language` values
- **The `outputType`** you requested
- **Host OS and Docker version** (`docker version`)
- **What you expected, and what happened instead** — including the exact error text or status code
- **Container logs** (`docker logs <container-id>`), redacted
- **A sample document**, if you are able to share one

> [!CAUTION]
> Never include your API key, license GUID, or confidential documents in a public issue. Redact them first. If reproducing a problem genuinely requires a sensitive document, say so in the issue and we will arrange a private channel.

## Billing and account questions

Subscriptions, upgrades, downgrades, invoices, payment methods, and cancellation are handled through the billing portal described in [API keys and plans](https://docs.abbyy.com/fine-parser/basics/api-keys-and-plans) — not through this repository. Enterprise customers are invoiced directly; contact your account manager.

## Interested in Enterprise?

Unlimited pages, orchestrated throughput of 500+ pages per second, fully air-gapped deployment, and engine tuning. It is in development now, and early conversations shape what ships — [contact us](https://www.abbyy.com/company/contact-us/) and tell us about your use case. See [FineParser Enterprise](https://docs.abbyy.com/fine-parser/reference/enterprise).

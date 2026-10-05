# 🚂 Express in Practice

An interactive Reveal.js presentation covering **Express in production** — the companion deck to *Introduction to Express.js*. Everything past "Hello World": project structure at scale, async error discipline, OpenAPI-first design, security hardening, observability, performance and graceful shutdown.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Express_in_Practice/)

## 📄 [Markdown Version](presentation.md)

## 🌟 [Foundation deck — Introduction to Express.js](https://brendanjameslynskey.github.io/Introduction_to_Express_js/)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | Pitch and the route → validate → authorise → handle → stream → observe flow |
| 02 | Topics | Map of foundations, behaviour, integration, production |
| 03 | Express 5 — what changed | Native promises, path-to-regexp v8, body parsers, breaking changes |
| 04 | Async error handling | One central handler, throw as control flow, process-level safety net |
| 05 | Project structure | Layered vs feature-based; the composition root |
| 06 | Validation at the boundary | Zod, Joi, TypeBox; coercion + defaults; one tool per repo |
| 07 | Authentication patterns | Sessions, JWT access/refresh, API keys, mTLS |
| 08 | Authorisation patterns | Route guards, resource-level checks, OPA/Cedar policy middleware |
| 09 | Streaming responses | Downloads, SSE, NDJSON, range requests, common pitfalls |
| 10 | File uploads | Multer, disk, direct-to-S3 signed URLs |
| 11 | Rate limiting | Global, per-user, per-endpoint; Redis store; abuse protection |
| 12 | Caching | Cache-Control, ETag/304, Redis cache-aside, stampede mitigation |
| 13 | Database integration | Repository + service shape, transaction-per-request, read replicas |
| 14 | Background jobs | BullMQ, idempotent workers, outbox pattern, dead-letter queues |
| 15 | Testing | supertest, auth helpers, contract tests against OpenAPI |
| 16 | Observability | pino structured logs, OpenTelemetry traces, RED metrics |
| 17 | Security hardening | helmet, CSP, CORS, CSRF, body limits, HSTS |
| 18 | Performance | Clustering, compression at the edge, serialisation, keep-alive |
| 19 | Graceful shutdown | SIGTERM sequence, liveness vs readiness, K8s tunables |
| 20 | Reverse proxy & TLS | trust proxy, nginx config, what the proxy keeps |
| 21 | OpenAPI & contract-first | Code-first vs spec-first, /openapi.json, SDK generation |
| 22 | Express vs Fastify, Koa, Hono, NestJS | Capability matrix and verdicts |
| 23 | Express 4 → 5 migration | Six-step playbook from environment bump to canary |
| 24 | Summary | Take-aways, checklist, further reading |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Speaker notes | `S` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Outfit + Plus Jakarta Sans + Fira Code

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

Express 5 migration — expressjs.com/en/guide/migrating-5.html · Node.js Best Practices — github.com/goldbergyoni/nodebestpractices · helmet — helmetjs.github.io · OpenAPI — openapis.org · JSON Schema — json-schema.org · OpenTelemetry — opentelemetry.io · BullMQ — docs.bullmq.io

## License

Educational use. Code examples provided as-is.

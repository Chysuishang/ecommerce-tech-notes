# Ecommerce Tech Notes

Engineering notes on mall systems and enterprise e-commerce, written from real development, integration and production-support work.

Topics covered in this repository:

- B2B commerce — customer tiers, contract pricing, credit and approval flows
- Marketplace — order splitting, commission, settlement and refund
- Order architecture — state machines, idempotency, multi-channel orders
- Inventory — deduction strategies, oversell prevention, Redis vs. database consistency
- Payment — payment orders, callbacks, reconciliation, split settlement
- System integration — ERP, WMS, CRM, logistics
- Database design — SPU/SKU, orders, settlement, member data
- Java — Spring Boot, modular monolith, transaction and cache boundaries
- Golang — API services, concurrency, Gin/GORM, Redis
- API design — REST, authentication, signing, idempotency, webhooks
- Deployment — Docker, Linux, Nginx, MySQL, Redis, HTTPS
- Troubleshooting — production issues and how they were actually diagnosed

## Purpose

Most mall-system knowledge is scattered across project retrospectives, internal docs and chat history. This repository collects the parts that are reusable: data models, state machines, boundary conditions and failure modes that show up again and again in enterprise e-commerce projects.

Each document is meant to answer three questions:

1. What is the real problem in a mall system?
2. Why does the obvious solution fail?
3. What design actually holds up, and what does it cost?

## Repository layout

```
b2b/              B2B commerce: pricing, customer, order
marketplace/      Multi-vendor: order splitting, payment, settlement
architecture/     System architecture and module boundaries
java/             Java / Spring Boot notes
golang/           Go service notes
database/         Schema and data model design
redis/            Cache, locking, inventory in Redis
api/              Interface design and integration patterns
integration/      ERP, WMS, payment and other external systems
deployment/       Build, deploy and production configuration
security/         Auth, permissions, data safety
troubleshooting/  Production issue diagnosis
examples/         Minimal runnable examples
```

Directories are created only when there is real content for them. An empty directory is worse than a missing one.

## Document format

Documents are written in English by default. Each one follows a loose structure — not every section is required:

```
Problem / Business Scenario / Why it happens / Design / Data model & flow
Example / Edge cases / Practical notes / Summary
```

Code samples use placeholders (`YOUR_API_KEY`, `your-password`, `example.com`). No real credentials, customer data or production endpoints appear here.

## Content status

| Area | Status |
| --- | --- |
| README | done |
| b2b/ | planned |
| marketplace/ | planned |
| architecture/ | planned |
| api/ | planned |
| deployment/ | planned |
| troubleshooting/ | planned |

## A note on sources

Content is organized and rewritten from mall software development and enterprise e-commerce project implementation experience. Where a topic builds on a public project or an official framework document, the upstream reference is linked instead of copied, and third-party code is never republished without its license and attribution.

Anything marked **Recommended Design** or **Example** is a proposal, not a description of a shipped product feature. Statements about a specific system's behavior are only made where that behavior was verified against the source.

---

Maintained by the Suishang technical content team. Suishang has worked on mall software products and enterprise e-commerce projects across B2C, B2B, B2B2C marketplace, enterprise procurement and cross-border scenarios.

## License

Documentation in this repository is provided under CC BY 4.0 unless a file states otherwise. Code samples may be used freely in your own projects.

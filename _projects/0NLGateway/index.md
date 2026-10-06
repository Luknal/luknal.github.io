---
layout: post
title: Natural-Language Gateway for 93 Data APIs
description: A Spring Boot service that turns a plain-language question into a call against the right one of 93 declared data APIs, with no code written per endpoint. Routing runs as a four-stage pipeline - keyword or vector retrieval, LLM reranking, LLM parameter extraction, then the HTTP call - over a tool catalogue defined entirely in YAML. Packaged with a dependency-free mock data service so the whole system runs locally.
skills:
- Java 17
- Spring Boot 3
- Spring AI
- LLM tool routing
- REST API design
- Docker
- Python
main-image: /chat.png
---

## Overview

An engineering team had production data spread across dozens of internal APIs and no way to ask a
plain question of any of it. Answering *"which ten wells produced the most today"* meant knowing
which endpoint to call and what parameters it took.

This service closes that gap. A question goes in, the right API gets called, and the answer comes
back as a labelled table — with no handler written per endpoint.

The screenshot above is that working: the question *今天产量最高的10口井* went in, and the assistant
called the production-ranking endpoint and labelled the result. Those column headings — 井号, 日产油,
含水率 — are not hardcoded anywhere in the application. They come from the `output_fields` block of
whichever tool the router picked.

The interesting constraint is that **adding an API is a configuration change, not a code change**.
All 93 endpoints are declared in YAML.

---

## How routing works

Four stages, two of which are LLM calls:

| Stage | What it does | Why it exists |
|---|---|---|
| Keyword / vector match | 93 tools → ~5 candidates | Cheap recall across the whole catalogue |
| LLM rerank | 5 candidates → 1 tool | Precision where the cheap stage is ambiguous |
| LLM parameter extraction | question → `{rq, limit, order_by}` | Turns prose into typed arguments |
| HTTP call + format | calls the endpoint, labels the columns | Declared in the tool's YAML entry |

Splitting retrieval into a cheap stage and an expensive one is the central design decision. Pure
vector search confuses *产量趋势* (output trend over time) with *产量排名* (ranking at a point in
time) — semantically adjacent, operationally different. But handing all 93 tool descriptions to a
model in one prompt is slow, costly, and gets worse as the catalogue grows. Cheap retrieval for
recall, expensive reranking for precision, over a candidate set small enough to fit in a prompt.

In production the first stage is bge-m3 embeddings against a Milvus vector store. The published
repo substitutes keyword matching so it runs without an embedding service.

![The four routing stages, with the two LLM calls highlighted](/_projects/0NLGateway/pipeline.png)

*Stages 2 and 3 are the only LLM calls; everything else is declarative.*

---

## The tool catalogue

Each API is a YAML entry — endpoint, parameters, and a description written for the model to read:

```yaml
- name: 查询产量排名
  endpoint: /api/analysis/top_wells
  method: GET
  description: |
    查询产量排名
    - 典型问题："今天产量最高的10口井"
  params:
    - name: rq
      description: 日期
      required: true
    - name: limit
      description: 返回数量
      required: false
  response:
    output_fields:
      - key: jh
        label: 井号
      - key: rcyou
        label: 日产油
```

That `description` is load-bearing: it is what both the matcher and the reranker read. Tuning
routing accuracy means improving a description rather than editing a classifier.

![The tool catalogue in the admin UI: base URL, endpoint, method, description and parameter count per tool](/_projects/0NLGateway/tool-catalogue.png)

Catalogue entries are managed in the admin UI rather than edited as files, and re-indexing them for
vector search is a button. Adding an API to the assistant's repertoire never touches Java.

The tradeoff is real and worth stating. When routing is driven by text, failures stop being crashes
and become *quietly wrong answers* — a badly worded description sends a question to a plausible but
incorrect endpoint. That is what pushed the production system toward citations and an answer
self-check: text-driven routing has to show its work.

---

## Making it run on someone else's machine

A gateway is useless to a reader who cannot run it, and this one talks to internal production APIs.
So it is packaged with `mock-api` — a data service in ~430 lines of standard-library Python, no
dependencies — serving 15 oilfield endpoints with synthetic data that is:

- **Deterministic** — values are hashed from the date and well number, so the same request always
  returns the same answer.
- **Internally consistent** — wellhead output, station metering, sales and tank volumes balance to
  within a ~0.5% loss rate.
- **Seeded with an anomaly** — eight to eleven days back the loss rate climbs to ~4%, one pipeline's
  pressure drops, and patrol logs mention a suspected illegal tap.

The last point matters most. Mock data that merely has the right shape can only prove a chart
renders. Mock data with a planted anomaly can prove the *analysis* works — that the detection logic
finds the thing it was built to find.

---

## Honest limitations

- **It needs an LLM to be useful.** Without a key the service starts and runs the pipeline
  end-to-end, but reranking and parameter extraction are both model calls — so it picks the first
  candidate and extracts nothing. Any OpenAI-compatible endpoint works, including a local model.
- **The published repo matches on keywords, not vectors**, using a hardcoded term list. It is a
  stand-in that exercises the same pipeline shape without requiring an embedding service and a
  vector database; it does not generalise beyond this domain.
- **Citations and the answer self-check are not included** — they live in the production system,
  which is part of a multi-tenant application that cannot be open-sourced.

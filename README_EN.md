# KSASNT

[中文](README.md) · English

A local, context-engineered knowledge engine. Ingest md / txt, prefilter by the question, chunk with overlap, extract, merge, and answer **with citations**.

**Source code is not published.** Binaries, if any, live on [Releases](../../releases) only.

![KSASNT product still: cited document Q&A](docs/images/01-hero.jpg)

## Pipeline

![Six-step path from ingest to cited answer](docs/images/02-pipeline.jpg)

Ingest → prefilter → overlapping chunks → extract → integrate → answer with `[doc:start-end]`. A separate faithful-original path skips abstractive extract and answers on relevant raw spans.

## Modes

![Precise query, explained Q&A, adaptive](docs/images/03-modes.jpg)

`query` for facts, `qa` for explanations, `auto` when you do not want to pick. Prompts live in local config.

## Source policy

No `.py` in this repository. Do not redistribute the installer; share the Releases URL. See [LICENSE](LICENSE).

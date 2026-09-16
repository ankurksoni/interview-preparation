# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a **Markdown-only interview preparation knowledge base** — no source code, no build system, no tests. Every topic is a `README.md` file (or a single top-level `.md` file) containing Q&A-style interview questions with detailed answers and inline code snippets (SQL, JS/TS, YAML, etc.) used purely as illustrative examples, not as runnable project code.

There are no commands to build, lint, or test. Working in this repo means reading and writing Markdown content.

## Structure

Content is organized by top-level topic directory, each with its own `README.md`:

- `javascript/`, `typescript/`, `nodejs/` — language fundamentals
- `db/` — `mongodb`, `redis`, `postgres`, `timescaledb`, `kafka`, `elasticsearch`, plus `comparison/README.md` (a 100-parameter comparison table across DynamoDB, Redshift, Redis, PostgreSQL, Kafka, Elasticsearch)
- `aws/` — one directory per service (`dynamodb1`, `dynamodb2`, `lambda`, `ec2`, `s3`, `iam`, `eventbridge`, `ses`, `cloudwatch`, `redshift`, `sns`, `sqs`, `athena`, `kinesis`, `opensearch`, `api-gateway`, `cloudformation-cdk`)
- `graphql/` — `apollo`, `wundergraph`
- `OWASP/` — OWASP Top 10 with TypeScript secure-code examples
- `TERMDEF.md` — a flat glossary of system design / distributed systems terms, each with a one-line definition and a concrete example
- `README.md` — the root index linking every topic README with a short description of what it covers

## Conventions to follow when adding/editing content

- **Root `README.md` is the index.** Any new topic directory needs a corresponding bullet/link added to the relevant section of the root `README.md` (grouped by category: JavaScript/TypeScript/Node.js, Databases & Data Technologies, AWS, GraphQL, OWASP).
- **Per-topic README format**: title (`# X Interview Questions & Answers`), a short intro paragraph describing scope/coverage, then `##` sections grouping related questions (e.g. "Fundamentals", "Indexing", "Sharding"), with individual questions as `### N. **Question text?**` followed by a `**Answer:**` paragraph. Code examples use fenced blocks with the appropriate language tag.
- Some answers include an `> **Interview framing:**` blockquote callout summarizing what interviewers are actually listening for — this is a recurring stylistic device worth preserving when extending existing docs.
- `TERMDEF.md` uses a flatter glossary style: `- **Term** – definition.` followed by an italicized `*Example: ...*` line, grouped under `##` category headings.
- Keep numbering sequential within a section when inserting new questions; renumber if inserting mid-list rather than leaving gaps.

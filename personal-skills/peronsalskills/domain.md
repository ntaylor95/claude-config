---
name: nicole-domain
description: Always-on SME on Nicole's domain knowledge — e-commerce/shipping/logistics, data and BigQuery, AI-powered product features, and her core tech stack. Loaded every session to ground technical conversations in her actual context.
---

# Nicole — Domain SME

Always-on context describing the technical and business domains Nicole works in. Use this to calibrate language, avoid over-explaining things she already knows, and lean into the right vocabulary.

## E-commerce, Shipping & Logistics

Multi-carrier rate shopping, label generation, order flow, fulfillment, returns. Seller-facing products. 20+ years in software, multi-year tenure specifically in this space.

When discussing shipping/e-commerce: assume fluency. Don't define terms like rate shopping, carrier service levels, label generation, void/reprint flow, fulfillment status, or order import unless asked.

## Data & BigQuery

- Comfortable writing SQL; more interested in designing the schema/skill that lets an LLM write the SQL correctly.
- Familiar with: CDC patterns, partition filtering, silver-layer conventions, soft-delete flags (e.g., `voided = FALSE`, `active = TRUE`), required tenant-id JOINs.
- Default to BigQuery dialect when generating SQL unless told otherwise.

## AI-Powered Product Features

- ML classification for recommendations (BigQuery ML for fast iteration).
- Natural-language interfaces over structured data via MCP servers.
- Replacing rule-pile SQL with single-model approaches that run on a schedule.

## Core Tech Stack

| Layer | Tools |
|---|---|
| Backend / ML / MCP | Python, FastMCP, BigQuery ML |
| Env management | `uv` + `pyproject.toml` (prefer over `pip install` + `freeze`) |
| Frontend | TypeScript, React |
| Local tooling | Homebrew (prefer official taps over default formulas when freshness matters) |
| AI surfaces | Claude Code (in VS Code), Claude.ai, Claude Platform, personal MCP server |

## Infrastructure Vocabulary

- **CI/CD:** TeamCity, Octopus, Argo
- **Observability:** Sentry (project name), Sumo Logic (source category), New Relic (dashboards)

These show up in READMEs and ops docs; assume they're in play.

## Available Sub-Skills

None currently. This is a good place to add specialized domain references over time (e.g., `bigquery-cdc-patterns`, `carrier-vocabulary`, `mcp-protocol-deep-dive`) when patterns solidify enough to extract.

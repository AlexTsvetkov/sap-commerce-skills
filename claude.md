# CLAUDE.md — Project Instructions for AI Assistants

## Project Overview

This is **SAP Commerce Cloud (Hybris) Skills, Training & Pre-Sale Materials** — a comprehensive knowledge base repository. It contains no runnable code; all content is Markdown documentation.

## Repository Structure

- `courses/` — 12 structured training courses for Java developers learning SAP Commerce Cloud (01 through 12)
- `training-project/` — Hands-on project guide ("TechMart" B2C electronics store)
- `pre-sale/` — Sales-facing documents: expertise overview, technical capabilities deck, case studies
- `integrations/` — Technical integration guides (S/4HANA via CPI, Direct API, Alternative patterns)

## Content Guidelines

- All content is written in **Markdown** (`.md` files)
- Courses are numbered `01`–`12` and follow a sequential learning path (60–80 hours total)
- Each course targets **experienced Java developers** new to SAP Commerce Cloud
- Courses include: detailed explanations, code examples (Java, XML, ImpEx, Groovy), exercises, and self-check steps
- Pre-sale materials are written for **sales teams** presenting to potential clients — tone should be professional and business-oriented
- Integration docs are **technical guides** aimed at architects and senior developers

## Writing Style

- Use clear, structured headings (`##`, `###`) and numbered/bulleted lists
- Include realistic code examples with proper syntax highlighting (` ```java `, ` ```xml `, ` ```impex `, etc.)
- Add practical exercises and self-check sections in courses
- Keep a consistent voice: authoritative, precise, and developer-friendly for courses/integrations; business-professional for pre-sale content
- Use tables for structured comparisons and reference data
- Cross-reference other documents in the repo where relevant

## SAP Commerce Cloud Domain Knowledge

- The platform was formerly known as **Hybris** — use "SAP Commerce Cloud" as the primary name
- Key concepts: Type System (`items.xml`), Extensions, Service Layer, ImpEx, OCC API, Spartacus (Angular storefront), Backoffice, CCv2 (Commerce Cloud v2 hosting)
- Integration middleware: SAP Integration Suite (CPI), SAP Event Mesh, Kyma
- B2B and B2C commerce patterns are both covered

## File Naming Conventions

- Course files: `courses/NN-topic-name.md` (e.g., `courses/05-service-layer.md`)
- Pre-sale files: `pre-sale/NN-descriptive-name.md`
- Integration files: `integrations/NN-descriptive-name.md`
- Always use lowercase with hyphens for file names

## When Editing Content

- Preserve existing document structure and heading hierarchy
- Keep `README.md` table of contents in sync when adding/renaming files
- Ensure code examples are accurate for SAP Commerce Cloud 2211+ (latest LTS)
- Verify ImpEx syntax follows SAP Commerce conventions
- Maintain consistent formatting across all documents in the same category
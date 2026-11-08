# Overview

nand and poliav are two repos with one contract between them.

## nand

**Purpose.** Understand and explain how an agent works by building one from scratch.

**Approach.**
- Written in Go, standard library first.
- An agent loop with a turn limit.
- A Tool interface (name, description, JSON schema, handler, permission level) and permission gates.
- A neutral internal message type, with provider specific fields stored as opaque data and sent back untouched.
- Provider adapters over OpenAI compatible endpoints, starting with Gemini.
- No domain knowledge. Nothing here imports or mentions Canvas or ESPOL.

## poliav

**Purpose.** Put ESPOL's Aula Virtual (Canvas LMS) within reach of an agent and of a person.

**Approach.**
- A `canvas` package with an API client, token rotation and pagination.
- A CLI on top of that package.
- Tools that register in nand through the Tool interface.
- Read only first, plus one harmless write action behind a permission gate.
- Calculations such as weighted averages happen in code, never in the model.
- Course content is untrusted input.
- Tokens renew by creating a new one, storing it, then deleting the old one. Regenerating in place kills the old value at once.

## How they fit

- poliav imports nand. nand never imports poliav.
- The Tool interface is the only contract between them.
- One question travels from the person to the agent loop in nand, through a tool, to the Canvas client in poliav and on to the Aula Virtual.

## Why two repos

- The harness stays reusable for any domain.
- The work splits cleanly along the Tool interface.
- Keeping them apart proves the harness does not depend on Canvas.

Goal: present both at Semana CIAP, December 14 to 18, 2026.

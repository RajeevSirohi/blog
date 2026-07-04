---
title: 'Debugging a Silent Failure in a Custom Terraform MCP Server'
description: 'A binary plan file that vanished before apply could use it — and what it taught me about designing tools for AI agents.'
pubDate: '2026-07-04'
tags: ['infrastructure', 'ai-engineering']
draft: true
---

<!-- DRAFT — outline below, flesh out each section then set draft: false -->

## 1. The setup

<!-- 2–3 paras -->

- Why I built a Terraform MCP server for Claude Desktop: letting Claude drive init → validate → plan → apply against a local workspace
- Stack: brief mention of the server implementation and tool surface (tf_init, tf_validate, tf_plan, tf_apply)
- The promise: plan once, apply exactly that plan — the whole point of binary plan files

## 2. The bug

<!-- The hook — lead with the symptom -->

- `tf_apply` with `planFile` pointing at the binary plan from `tf_plan` fails *silently* — no error, no apply
- Why silent failures are worse than loud ones in agent tooling: the model believes the tool succeeded and reports success to the user
- Reproduction steps, exact tool calls, what the logs showed (and didn't)

## 3. The investigation

- Hypotheses tested: path resolution, working-directory mismatch, plan file staleness, argument passing
- How I narrowed it down — what ruled each hypothesis out
- Root cause: the plan tool wrote the binary plan to a temp file and deleted it in a `finally` block after returning the diff — so the path it printed was already gone by the time apply was called

## 4. The workaround

- Call `tf_apply` with only `workdir` + `confirmed: true` — reliable
- Trade-off being accepted: you lose plan-pinning; apply re-plans. When that's acceptable and when it isn't
- The actual fix: only write a plan file when the caller explicitly asks for one to persist

## 5. The bigger lesson

<!-- This is what makes it a good post, not just a bug report -->

- Agent-facing tools must fail loudly. Design rule: every tool result should be verifiable by the model
- Concrete design changes: apply returns the plan digest it executed; a missing plan file is a hard error, not a fallback

## 6. Close

- One-line summary + [link to the repo](https://github.com/RajeevSirohi/mcp-server-terraform)
- Corrections welcome — best early engagement comes from people telling you what you missed

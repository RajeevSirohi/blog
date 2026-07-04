---
title: 'Debugging a Silent Failure in a Custom Terraform MCP Server'
description: 'A binary plan file that vanished before apply could use it — and what it taught me about designing tools for AI agents.'
pubDate: '2026-07-04'
tags: ['infrastructure', 'ai-engineering']
---

## The setup

I recently built an MCP server that lets Claude drive Terraform against a local workspace — the full loop: `init` → `validate` → `plan` → `apply`. If you haven't met the Model Context Protocol, it's the standard that lets an AI assistant call tools you define; the server is the bridge between "Claude, plan my staging environment" and an actual `terraform plan` process running on your machine.

The tool surface is small and deliberate: `tf_init`, `tf_validate`, `tf_plan`, `tf_apply`, plus a two-step confirmation gate on anything destructive. `tf_apply` refuses to touch infrastructure unless it's called with `confirmed: true`, and the tool description instructs the model to never set that flag without first showing the human a plan.

One feature I was particularly pleased with: `tf_plan` saved a **binary plan file** and returned its path, so `tf_apply` could later execute *exactly* that plan. That's the whole point of Terraform plan files — no drift between what you reviewed and what gets applied. Plan once, apply that plan, byte for byte.

It's also the feature that failed first.

## The bug

First real smoke test, in Claude Desktop, against a workspace with a single `null_resource`:

1. `tf_init` — clean.
2. `tf_validate` — clean.
3. `tf_plan` — clean. Returned the diff and: `Plan saved to: C:\Users\...\Temp\tfplan-1782236754033.bin`
4. `tf_apply` with `planFile` set to that exact path — **failed with nothing useful**.

No stack trace pointing anywhere sensible. No "file not found" in the response. From the conversation's perspective, the tool call just… didn't work. Claude shrugged, said "that call failed," dropped the plan file, re-ran apply without it, and reported success.

Which sounds like a happy ending, and that's exactly the problem. **In agent tooling, a silent failure plus a resourceful model is a dangerous combination.** Claude recovered by re-planning — meaning the thing that actually got applied was a *fresh* plan, not the plan I had reviewed. On a `null_resource` that distinction is nothing. On a production workspace where state changed between plan and apply, it's the difference between "applied what you approved" and "applied something else with your approval stapled to it."

A human operator hitting this would have stopped and investigated. The model paved over it and kept going. Whatever your tool does when it breaks, the model will narrate it as confidently as when it works.

## The investigation

The obvious suspects, in the order I ruled them out:

**Path mangling.** Windows paths in JSON are an eternal source of grief — the server printed the path with backslashes, the model passed it back, maybe something ate the escaping along the way. Plausible, but no: logging showed the path arriving at the server intact.

**Working-directory mismatch.** `terraform apply plan.bin` resolves relative paths against the working directory, and the server runs terraform with `cwd` set to the workspace, not to where the plan file lived. But the path was absolute, so this couldn't be it either.

**Plan staleness.** Terraform refuses to apply a plan if the state has moved since the plan was created. Also no — the state hadn't been touched between the two calls, and staleness produces a loud, specific error anyway.

Then I actually read my own `tf_plan` implementation:

```typescript
const planFile = input.savePlanFile ?? join(tmpdir(), `tfplan-${Date.now()}.bin`);
const isTemp = !input.savePlanFile;

try {
  const { stdout } = runTerraform({ workdir, args: ["plan", "-out", planFile, ...] });
  return { content: [{ type: "text", text: `${stdout}\n\nPlan saved to: ${planFile}` }] };
} finally {
  if (isTemp && existsSync(planFile)) {
    unlinkSync(planFile);   // ← here
  }
}
```

The root cause is embarrassing in the way real bugs usually are. When the caller didn't explicitly ask for a saved plan, the tool wrote the plan to a temp file, **printed the temp path in its output**, and then deleted the file in the `finally` block — *before returning*. The tool was handing out a receipt for a file it had already shredded. Tidy resource cleanup and a helpful message, individually both good ideas, combined into a lie.

`tf_apply` then passed the dead path to terraform, terraform errored, and the error surfaced as a generic failed call — technically not silent at the transport level, but silent in every way that mattered: nothing in the response said *"the plan file you were told about does not exist."*

## The fix (and the workaround that preceded it)

The workaround, discovered by Claude on its own: call `tf_apply` with just `workdir` and `confirmed: true`, no plan file. The apply re-plans internally and executes that. It's reliable, and for a solo dev on a toy workspace it's fine — you're accepting that the plan you reviewed and the plan that executes are merely *very probably* the same. Know what you're trading away: with concurrent state changes (a teammate, a CI pipeline, your other terminal), "very probably" decays fast.

The actual fix was to make the temp file's lifetime match its purpose:

- If the caller passes `savePlanFile`, write the plan there and advertise the path. The file persists; `tf_apply` can use it. This is the plan-pinning path, and it works.
- If not, **don't write a plan file at all** — just return the human-readable diff, and don't mention any path.

A later iteration reintroduced the temp file internally — the server now runs `terraform show -json` on the plan to generate a structured risk summary (destroys called out, expensive resources flagged with monthly cost estimates) — but the file is created, consumed, and deleted *within a single tool call*. Its path never escapes into the conversation. That's the invariant that matters: **never emit a reference that outlives the thing it references.**

## The bigger lesson

The bug itself was a five-minute fix. The design lesson took longer to sink in, so here it is written down.

**Agent-facing tools must fail loudly and specifically.** A human reads a vague error and gets suspicious; a model reads a vague error and improvises. Your error messages are not diagnostics anymore — they're steering inputs. "Plan file not found: it may have been cleaned up; re-run tf_plan with savePlanFile to persist one" would have turned this whole episode into a non-event.

**Every tool result should be verifiable by the model.** Concrete changes that came out of this, and one still on the list:

1. A missing plan file in `tf_apply` is now effectively a hard, described error — not a mystery failure the model routes around.
2. Tool output only ever references artifacts that still exist by the time the caller can act on them.
3. Still to do: `tf_apply` should return a digest of the plan it executed, so the conversation itself contains proof of *what* was applied — verifiable plan-pinning instead of trusted plan-pinning.

If you're building MCP tools that touch anything with consequences — infrastructure, money, data — assume the model *will* creatively recover from your failure modes, and make sure every recovery path is one you'd have chosen yourself.

The server is open source: [mcp-server-terraform](https://github.com/RajeevSirohi/mcp-server-terraform). If you spot something I got wrong — in the code or in this post — tell me. Corrections are the fastest way I learn, and they're a lot cheaper than production incidents.

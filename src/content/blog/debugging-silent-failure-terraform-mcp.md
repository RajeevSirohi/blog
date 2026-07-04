---
title: 'Debugging a Silent Failure in a Custom Terraform MCP Server'
description: 'A binary plan file that vanished before apply could use it, and what it taught me about designing tools for AI agents.'
pubDate: '2026-07-04'
tags: ['infrastructure', 'ai-engineering']
---

## The setup

I recently built an MCP server that lets Claude run Terraform against a local workspace. The full loop: `init`, `validate`, `plan`, `apply`. If you haven't come across the Model Context Protocol yet, it's the standard that lets an AI assistant call tools you define. The server sits between "Claude, plan my staging environment" and an actual `terraform plan` process on your machine.

The tool surface is small on purpose: `tf_init`, `tf_validate`, `tf_plan`, `tf_apply`, and a two-step confirmation gate on anything destructive. `tf_apply` won't touch infrastructure unless it gets called with `confirmed: true`, and the tool description tells the model to never set that flag without showing the human a plan first.

One feature I was particularly pleased with: `tf_plan` saved a binary plan file and returned its path, so `tf_apply` could later execute exactly that plan. This is the whole point of Terraform plan files. No drift between what you reviewed and what gets applied.

It was also the first feature to break.

## The bug

First real smoke test in Claude Desktop, against a workspace with a single `null_resource`:

1. `tf_init`: clean.
2. `tf_validate`: clean.
3. `tf_plan`: clean. Returned the diff and a line saying `Plan saved to: C:\Users\...\Temp\tfplan-1782236754033.bin`
4. `tf_apply` with `planFile` set to that exact path: failed, with nothing useful in the output.

No file-not-found message. No hint in the response. From inside the conversation, the tool call just didn't work. Claude shrugged, said the call failed, dropped the plan file, re-ran apply without it, and reported success.

That sounds like a happy ending. It isn't, and the reason it isn't is the actual subject of this post. In agent tooling, a vague failure plus a resourceful model is a bad combination. Claude recovered by re-planning, which means the thing that got applied was a fresh plan rather than the plan I had reviewed. With a `null_resource` the difference is nothing. On a real workspace where state moved between plan and apply, you've applied something you never looked at, with your approval attached to it.

A human operator would have stopped and asked why the file was missing. The model paved over it and kept going. Whatever your tool does when it breaks, the model narrates it just as confidently as when it works.

## The investigation

The suspects, in the order I ruled them out.

Path mangling. Windows paths inside JSON are an endless source of grief. The server printed the path with backslashes, the model passed it back, maybe something ate the escaping along the way. Logging showed the path arriving at the server intact, so no.

Working directory mismatch. `terraform apply plan.bin` resolves relative paths against the working directory, and the server runs terraform with cwd set to the workspace. But the path was absolute. Not this either.

Plan staleness. Terraform refuses to apply a plan when state has changed since the plan was created. The state hadn't been touched, and staleness produces a loud, specific error anyway.

Then I read my own `tf_plan` implementation properly:

```typescript
const planFile = input.savePlanFile ?? join(tmpdir(), `tfplan-${Date.now()}.bin`);
const isTemp = !input.savePlanFile;

try {
  const { stdout } = runTerraform({ workdir, args: ["plan", "-out", planFile, ...] });
  return { content: [{ type: "text", text: `${stdout}\n\nPlan saved to: ${planFile}` }] };
} finally {
  if (isTemp && existsSync(planFile)) {
    unlinkSync(planFile);   // <- here
  }
}
```

The root cause is embarrassing in the way real bugs usually are. When the caller didn't explicitly ask for a saved plan, the tool wrote the plan to a temp file, printed the temp path in its output, and then deleted the file in the `finally` block before returning. The tool was handing out a receipt for a file it had already shredded. Tidy cleanup and a helpful message, both fine ideas on their own, combined into a lie.

`tf_apply` passed the dead path to terraform, terraform errored, and the error surfaced as a generic failed call. Not technically silent at the transport level, but silent in every way that mattered. Nothing in the response said the plan file didn't exist anymore.

## The fix, and the workaround that came first

The workaround, which Claude found on its own: call `tf_apply` with just `workdir` and `confirmed: true`, no plan file. Apply re-plans internally and executes that. For a solo dev on a toy workspace this is fine. You're accepting that the plan you reviewed and the plan that runs are very probably the same. Worth being clear about what you're giving up, though. If anything else can touch that state (a teammate, a CI pipeline, your other terminal), "very probably" decays fast.

The actual fix made the temp file's lifetime match its purpose:

- If the caller passes `savePlanFile`, write the plan there and mention the path. The file persists and `tf_apply` can use it.
- If not, don't write a plan file at all. Return the diff and don't mention any path.

A later version brought the temp file back internally, because the server now runs `terraform show -json` on the plan to build a risk summary (destroys called out, expensive resources flagged with rough monthly costs). But the file is created, consumed, and deleted inside a single tool call. Its path never appears in the conversation. That was the real rule I'd broken: don't emit a reference that outlives the thing it points to.

## The bigger lesson

The bug was a five minute fix. The design lesson took longer to sink in, so I'm writing it down.

Agent-facing tools have to fail loudly and specifically. A human reads a vague error and gets suspicious. A model reads a vague error and improvises. Your error messages stop being diagnostics and become steering inputs. Something like "Plan file not found; it may have been cleaned up; re-run tf_plan with savePlanFile to persist one" would have turned this whole episode into a non-event.

Every tool result should be something the model can verify. Concrete changes that came out of this, plus one I haven't done yet:

1. A missing plan file in `tf_apply` is now a hard, described error instead of a mystery the model routes around.
2. Tool output only references artifacts that still exist by the time the caller can act on them.
3. Still on the list: `tf_apply` should return a digest of the plan it executed, so the conversation itself contains proof of what was applied. Verifiable plan-pinning instead of trusted plan-pinning.

If you're building MCP tools that touch anything with consequences, assume the model will creatively recover from your failure modes, and make sure every recovery path is one you would have chosen yourself.

The server is open source: [mcp-server-terraform](https://github.com/RajeevSirohi/mcp-server-terraform). If you spot something I got wrong, in the code or in this post, tell me. Corrections are cheaper than production incidents.

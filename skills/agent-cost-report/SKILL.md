---
name: agent-cost-report
description: Reports what AI coding-agent sessions have cost on this machine, grouped by repository, by model, or by the client you bill, with a markup applied. Use this skill when the user asks what Claude Code or Codex has cost them, which project or client is responsible for their AI spend, what to put on a client invoice for AI usage, why their agent bill is high, or which sessions were the expensive ones. Also use it when they want to map repositories to clients for billing, or ask whether a particular repo is worth what it costs.
---

# Agent cost report

Prices the Claude Code and Codex CLI session logs that already exist on this machine, and reports
what they cost — per repository, per model, or per client being billed.

Everything here runs locally. It reads two directories that already exist and makes exactly one
network request, a `GET` for a public per-model price table. No transcripts, code, diffs, tool output
or environment variables are sent anywhere, and no account is required.

## When to use this skill

- "What has Claude Code cost me?" / "What am I spending on agents?"
- "Which project is eating my AI budget?"
- "What do I bill Acme for AI usage this month?"
- "Why was last month so expensive?" / "Which sessions cost the most?"
- "Can I charge my client for this?"
- The user is writing an invoice and needs an AI usage line.

## How to run it

The report:

```sh
npx -y tallyhook@0.5.1 --json                       # last 30 days, grouped by repository
npx -y tallyhook@0.5.1 --days 7 --json              # any window
npx -y tallyhook@0.5.1 --by model --json            # grouped by model
npx -y tallyhook@0.5.1 --by client --json           # grouped by the clients being billed
```

Mapping repositories to clients, which is what makes a billable figure possible:

```sh
npx -y tallyhook@0.5.1 clients                      # the current map, and repos no client claims
npx -y tallyhook@0.5.1 clients add "Acme Corp" acme-web acme-api --rate 20
npx -y tallyhook@0.5.1 clients rm "Acme Corp"
npx -y tallyhook@0.5.1 clients markup 20            # default rate for clients without their own
```

A pattern is matched case-insensitively as a substring of the repository string, which is
`github.com/org/repo` when there is a git remote and `local:<folder>` when there is not. A `*` makes
it an anchored glob instead. The map is stored in `~/.tallyhook/clients.json` and is never uploaded.

Prefer `--json` when you are going to reason about the numbers, and the plain table when you are
showing the user output directly.

## Reading the output correctly

Four things decide whether your answer is right or misleading.

**These are list-price equivalents, not a bill.** Every figure is what the same tokens would cost at
the provider's published API rate. On a Claude Max, Team or ChatGPT subscription there is no per-token
charge at all, so the number is the value of what was consumed, not money that left an account. Say so
whenever you quote a total.

**If `priced` is `false`, do not give a dollar figure.** The price table was unreachable and every
cost is `null`. Report token counts and say prices were unavailable. Never estimate the gap yourself.

**A client report deliberately does not cover the whole machine.** It counts only repositories that
someone has mapped to a client. The JSON carries `counted` and `not_counted_cost_usd`, and those two
always add up to `total_cost_usd`. If you quote a client total, check `not_counted_cost_usd` before
implying it represents everything — a half-mapped machine produces a real number for a partial
picture, which is easy to present as if it were complete.

**Billable figures are rounded per row, then summed.** That is deliberate: a client adding up the
lines you showed them has to arrive at the total you billed. Do not re-derive the total from the
unrounded costs.

**`unpriced_models` lists any model with no known price.** Those are counted as zero, so a non-empty
list means the total is an understatement. Mention it rather than letting it pass silently.

## Answering well

Lead with the total for the window and whether it is up or down against the previous equally long
window, then the two or three repositories or clients responsible for most of it. Point out anything
genuinely worth acting on: one repository dominating, a single runaway session, or most of the spend
sitting on an expensive model where a cheaper one would plausibly do.

If the user asks about billing a client and no clients are mapped, do not invent a grouping — show
them `clients add` and let them decide what belongs to whom. Never guess a markup; if they have not
said what they charge, ask, or add the client with no rate so the billable figure equals cost.

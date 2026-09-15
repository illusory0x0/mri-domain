# Fast lane

For quick prompts, pick the cheapest skill that still answers the question.

## The one rule

**Conversation first.** If the fact is already in the conversation — an earlier
tool output, a file already read, a decision already stated — use it. Do **not**
re-read the codebase to re-derive it. Re-reading is the dominant cost in a quick
prompt, and it is pure waste when the answer is already in context.

Only when the fact is genuinely absent do you go look, and then do a **single,
targeted read**, not a broad scan. When the missing context has to cross a
session boundary, that is what `/handoff` is for.

## Three axes

- **Conversation share** — how much the skill works from context already in the
  window (high = cheap).
- **Codebase read** — how much it must read or scan the repo (none / light /
  heavy).
- **Sub-agent tax** — whether it dispatches sub-agents. Under a throughput-bound
  backend (slow tok/s) this is what decides wall-clock time.

Fast-lane fit follows from the three.

## The matrix

| Skill | Conversation | Codebase read | Sub-agent | Fast prompt |
|---|---|---|---|---|
| `/ask-me` | high | none | no | ✅ |
| `/wait-what` | high | none | no | ✅ |
| `/ask-matt` | high | none | no | ✅ |
| `/to-questionnaire` | high | none | no | ✅ |
| `/handoff` | high | none | no | ✅ |
| `/codebase-design` | high | none | no | ✅ |
| `/find-skills` | medium | none (external) | no | ✅ |
| `/domain-modeling` | high | light | no | ✅ |
| `/grilling`, `/grill-me` | high | medium (per-fact lookup) | **yes** | ⚠️ disable lookup, or use `/ask-me` |
| `/grill-with-docs` | high | medium+ (CONTEXT/ADR) | **yes** | ⚠️ |
| `/to-spec` | medium | heavy (find seams) | no | ❌ |
| `/to-tickets` | medium | heavy | no | ❌ |
| `/code-review` | low | heavy (diff × two agents) | **yes ×2** | ❌ |
| `/implement` | low | heavy | no | ❌ |
| `/tdd` | low | heavy | no | ❌ |
| `/diagnosing-bugs` | low | heavy (loop-building) | no | ❌ (but worth it) |
| `/triage` | medium | heavy | no | ❌ |
| `/wizard` | medium | heavy (.env, CI) | no | ❌ |
| `/prototype` | medium | light→heavy + artifact | no | ❌ |
| `/resolving-merge-conflicts` | low | heavy | no | ❌ (when needed) |
| `/research` | low | external | **yes** (background) | ❌ (non-blocking) |
| `/improve-codebase-architecture` | low | very heavy (whole repo) | **yes** | ❌ don't reach casually |
| `/wayfinder` | medium | heavy (multi-session) | **yes** | ❌ |
| `/teach` | low | light + artifact | no | ❌ beginners only |

## The four tiers

**Fast lane — closed over the conversation.** No codebase read, no sub-agent.
Safe for any quick prompt:

`/ask-me`, `/wait-what`, `/ask-matt`, `/to-questionnaire`, `/handoff`,
`/codebase-design`, `/find-skills`, `/domain-modeling`.

**Middle — conversation-driven, but taxed by per-fact lookups.** The interview
runs on context, but `/grilling` sends a sub-agent to the environment for every
fact. Fast **only** once lookup is off — reach for `/ask-me` instead, or run
`/grilling` with lookup disabled:

`/grilling`, `/grill-me`, `/grill-with-docs`.

**Heavy — codebase-driven.** Worth it when the question genuinely needs the
repo. Ask the question first; don't reach for these to *find* it:

`/implement`, `/tdd`, `/diagnosing-bugs`, `/to-spec`, `/to-tickets`,
`/code-review`, `/triage`, `/wizard`, `/prototype`,
`/resolving-merge-conflicts`.

**Deep water — the heavy extreme: multi-session, whole-repo, or a large
artifact.** Reserved for exactly their case:

`/research`, `/improve-codebase-architecture`, `/wayfinder`, `/teach`.

## Quick substitutes

- `/code-review` → review the diff yourself in the main context, one axis; split
  into two sub-agents only when the change is large.
- `/improve-codebase-architecture` → ask for the **text candidate list** first;
  render the HTML only after you've picked one.
- `/grilling` → use `/ask-me` when the facts are already in the conversation.
- `/implement` → skip the `/code-review` tax on a trivial ticket.

## When to hand off

`/handoff` is narrow, and it is not a routine step. You need it only when:

- swapping to a new harness,
- moving to a new directory or repo,
- sending the work to a colleague,
- or forking a side task mid-phase.

What it buys is **portability**: it carries the context so the next session does
not have to re-read the codebase to get it back. See
[PHASE-BOUNDARIES.md](PHASE-BOUNDARIES.md) for the full tree.

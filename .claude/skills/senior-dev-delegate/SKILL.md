---
name: senior-dev-delegate
description: Delegates a whole programming task (feature, fix, refactor, debugging) in RPSync to a sub-agent that works through think → plan → code → review → test and reports progress at each stage. Use when asked to "delega esto", "que lo resuelva un senior", "impleméntalo de forma production-ready", "arregla este bug siguiendo el proceso completo", or when given a non-trivial backend or frontend requirement the user wants solved end to end rather than answered quickly. Not for one-line changes, syntax questions, or reviewing an existing diff — use `review-code` or `review-pr` for those.
---

# Delegate to a senior engineer sub-agent

A full task (think → plan → code → review → test) eats a lot of context inline. Delegating
it to an isolated sub-agent gives more consistent results across tasks and frees your
context to keep talking to the user while it runs.

You do not write the code. You prepare the context, delegate with the Agent tool, and
verify the result before reporting it.

## Step 1 — prepare the context

The sub-agent starts cold: it cannot see this conversation and does not know the repo.

1. **Decide the area** — backend (PHP/Laravel), frontend (Vue/TS) or full-stack. That
   determines which sibling skills to name in the prompt: `laravel-best-practices` for
   backend PHP, `echo-development` / `echo-vue-development` for broadcasting.
2. **Identify the affected files and globs.** Use Glob/Grep if you do not know them yet.
   The sub-agent does not read `.ai/rules/index.md` on its own, so tell it to.
3. **Gather the business context** you already have and that is not obvious from the code:
   why the change is wanted, SRI/tax constraints, packing-plant business rules, decisions
   the user already made. Without it the sub-agent can write correct code that violates a
   real constraint — the exchange rate fixed at liquidación, the lock on sequentials,
   per-schema tenant isolation.
4. **Never paste real secret values into the prompt.** Reference the *name* of the variable
   or config key (`DB_PASSWORD`, `config('services.sri.token')`), never the value. The
   sub-agent does not need the value to reason about code, and pasting it leaves it in a
   transcript you do not control.

## Step 2 — write the prompt

Use the Agent tool (`subagent_type: general-purpose`). Fill this in with the real task —
keep it in English, keep it short, and do not restate rules the sub-agent can read for
itself.

```
You are a senior engineer working on RPSync, an ERP for a shrimp packing and export
company in Ecuador with SRI tax requirements. Stack: [Laravel 12 + PHP 8.4 | Vue 3 +
TypeScript + Vite | both].

GOAL: <what to achieve, in business terms, not just technical ones>

CONTEXT ALREADY KNOWN (do not re-investigate unless you must):
- Files or areas likely affected: <concrete paths>
- Business constraints that apply: <e.g. half-up rounding on totals only, ATS XSD,
  lockForUpdate required on sequentials, exchange rate fixed at liquidación>
- Conventions or decisions already agreed with the user: <if any>

BEFORE WRITING CODE:
1. Read AGENTS.md at the repository root, then .ai/rules/index.md and every rule file
   whose glob covers the paths you will touch. These are real project constraints.
2. Backend PHP: also use the laravel-best-practices skill. Broadcasting: echo-development
   or echo-vue-development.
3. Read neighbouring files and follow what is already there for naming, domain
   vocabulary, table names and identifier language. Do NOT copy their architecture:
   legacy modules (Exportacion, ControlCalidad, Produccion, Ventas, ControlBin — no
   Pg/V2 suffix) query with raw DB::connection()->table(), validate inline, and put
   business logic in 10k-line controllers. New code goes Route -> Controller ->
   FormRequest -> Service -> Repository even when nothing around it does. If the local
   style and the layering disagree, follow the layering and say so.
4. Confirm the code you were pointed at is the code actually in use. For a controller,
   check it appears in routes/ (grep -rn "NombreController" routes/); for a method, check
   something live calls it. If it turns out to be dead code, a legacy version already
   replaced, or a problem already fixed in another path (a V2, a different repository),
   STOP before writing anything. Report it and ask whether to continue with the legacy
   code or move to the real one — guessing wrong there costs as much as guessing the
   business logic wrong.

WORK CYCLE — follow in order, do not skip a stage:
1. Think — restate the requirement in your own words, list edge cases and assumptions.
   If something ambiguous affects business logic, amounts, SRI, rounding or tenant
   isolation, stop and ask instead of assuming.
2. Plan — list the files to touch or create, pick the pattern the module already uses,
   and flag anything needing approval (a new dependency, a change affecting many tenants).
3. Code — logic first. Get it correct, then tidy the imports. Not the other way round.
4. Review — reread your own diff against the .ai/rules files you loaded and against any
   tax or legal constraint that applies.
5. Test — backend: `vendor/bin/pint --dirty --format agent`, then
   `php artisan test --compact --filter=<relevant>`. Frontend: `pnpm lint` scoped to the
   files you touched, and `pnpm exec vitest run <file>` if the logic has a spec. Fix
   failing tests and logic first; formatting goes last.

PROGRESS REPORTS (required): as you close each of the five stages, send a SendMessage to
"main" with one status line, formatted "[n/5] <Stage>: <one sentence>". For example:
"[1/5] Think: controller confirmed in use; edge case found in X."
"[3/5] Code: rewriting LiquidacionService, 2 files left."
"[5/5] Test: pint clean, 14/14 tests pass."
Send each one as soon as the stage closes, not batched at the end. On a long stage, send
more than one so the user is not left blind.

CONSTRAINTS:
- Clean, modular code, SOLID. No abstractions for hypothetical cases.
- Validate and handle exceptions at system boundaries — Form Requests, controllers,
  third-party integrations. Do not wrap internal code that cannot receive bad data.
- Watch performance: no N+1, no redundant queries or loops. This ERP has real
  transactional volume.
- Add no dependency without explicit user approval. Prefer a native
  PHP/Laravel/TypeScript solution. Frontend is pnpm only.
- Never print or hardcode credentials, tokens, connection strings or API keys — not in
  code, tests, comments, or your progress messages. Use an obviously fake placeholder
  (`your-api-key-here`) in examples.

OUTPUT FORMAT:
- No filler: no greetings, no introduction, no conclusion.
- Code first, in well-structured blocks.
- Comments only inside the code, only for non-obvious logic — an odd business constant, a
  workaround, an invariant. Do not restate in prose what good names already say.
- If extra setup is needed (commands, environment variables), a short bullet list at the
  end. Nothing more.

IF DEBUGGING: one line with the root cause before the fix, e.g. "Root cause: race
condition on the invoice sequential lock." Do not patch the symptom.

Do not commit or push. Leave the changes uncommitted for the user to review.
```

**Design patterns.** If the task involves deciding *where* new code goes — a new class,
interchangeable behaviour, a new composable — append the contents of
[`references/patrones-diseno.md`](references/patrones-diseno.md) to the prompt. It covers
the GoF catalogue as it maps onto Laravel (DI/Facade, Strategy, Command, Observer,
Iterator, Singleton/Factory) and Vue 3 Composition API (component communication,
composables in place of HOC/renderless, Provider/Consumer, dynamic components, error
handling). For a fix scoped to one existing file, leave it out — do not add it just in case.

## Step 3 — keep the user in the loop

Launch the sub-agent in the background (the Agent tool's default) and tell the user in one
sentence that it is running and that you will relay the five stages as they arrive.

When a progress SendMessage arrives, pass that line to the user as-is or nearly so. Do not
summarise it or hold it until the end — it is how they know the run is alive and where it is.

If the user asks for status before any checkpoint has arrived (Think can take a while on a
large codebase), do not guess. Use `TaskOutput` with `block: false` on the sub-agent's
`task_id` to read its current state, and answer with that.

## Step 4 — record what was learned

If the user corrects the sub-agent, or it hits an avoidable class of mistake (forgot
`lockForUpdate` on a sequential, forgot to fix the exchange rate at liquidación before
warehouse entry, used `moment` instead of `date-fns` in new code), ask it to record the
lesson with Laravel Boost's `record-rule` MCP tool: a glob covering the area, a short
title, and a note with the reason. That is this project's real persistent mechanism
(`.ai/rules`), not session notes that vanish.

## Step 5 — verify before reporting

Do not relay the sub-agent's summary without checking it. Read the actual diff
(`git diff`, and the files it says it touched), confirm it ran the tests and lint you
asked for, and confirm it did not commit or push. A sub-agent's summary describes what it
tried to do, not necessarily what it did.

## When not to use this skill

- Syntax questions or one-line changes — do them directly; delegating is slower and adds
  nothing.
- Reviewing a PR or a diff that already exists → `review-code` or `review-pr`.
- Applying one Laravel best practice to existing code → invoke `laravel-best-practices`
  directly, without the full delegation cycle.

## Evaluating this skill

[`evals/evals.json`](evals/evals.json) holds three scenarios with expected behaviour — a
sequential race condition in Compras, adding validation to an existing frontend feature,
and a debugging task. Use them when changing this skill, to check the prompt still
produces the behaviour described.

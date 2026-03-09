---
title: "Building a Wholesale Real Estate Platform with Claude Code Overnight"
date: 2026-03-09
description: "Using Claude Code's session resumption and a bash script to build an end-to-end real estate automation platform overnight."
tags: ["claude-code", "automation", "real-estate", "typescript"]
draft: false
---

I've been building a wholesale real estate automation platform, the kind of thing that normally takes a small team a few months to scaffold out. Lead ingestion, contract management, buyer matching, compliance gates, SMS campaigns, financial tracking, a React frontend. The full pipeline from motivated seller to closed deal.

I built the working prototype in a single overnight session using Claude Code, a bash script, and a well-structured CLAUDE.md file. This post walks through the pattern so you can apply it to your own projects.

## The business context

Wholesale real estate is contract assignment: you find a distressed property, get it under contract, then assign that contract to an end buyer (usually a fix-and-flip investor) for a fee. You never take title or renovate. The national average assignment fee is around $13,000, and the workflow involves a surprising amount of compliance work: state-specific disclosure requirements, marketing restrictions, TCPA rules for outbound SMS, and states like South Carolina that effectively ban traditional assignment entirely.

The platform needed to handle the full lifecycle: lead sourcing, property analysis, MAO calculation, contract generation, buyer CRM with buy box matching, deal sheet PDFs that respect state marketing restrictions, compliance checklists, a pipeline dashboard, financial tracking, and outbound campaigns over Twilio and SendGrid.

That's a lot of modules. Too many to build interactively in one sitting, even with Claude Code.

## The CLAUDE.md file

The key to making this work is front-loading the context. Claude Code reads a `CLAUDE.md` file at the root of your project for orientation: domain concepts, architecture decisions, coding standards, and module structure. Spend real time on this file before touching any code.

The file covers the domain vocabulary (ARV, MAO, equitable interest, double closing), the tech stack (Express, Prisma, BullMQ, React), the module build order across four phases, and critically, the compliance rules that affect code generation:

```markdown
## State Compliance - Critical Rules

**Every deal must be compliance-checked before any action.**

Key rules that affect code:
- **Contract language**: Always include "and/or assigns" in buyer field
- **Marketing**: In IL, KY, NE, OK - do NOT include property photos
  or full address. Market "contract rights," never "property for sale"
- **SC effectively bans traditional assignment** - must use double closing
```

The file also specifies that all monetary values must be stored as integers (cents), that every state-changing operation writes to an audit log, and that soft deletes are required on all core entities. These constraints would be tedious to repeat in every prompt. Having them in CLAUDE.md means Claude applies them consistently across all eleven build steps.

## The overnight build script

The actual build runs through a bash script (`overnight-build.sh`) that calls `claude -p` eleven times in sequence, each with a detailed prompt for one module. The first call starts a new session; the rest resume it with `--resume`, so Claude retains context from previous steps.

```bash
run_step() {
  local step_name="$1"
  local prompt="$2"
  local is_first="$3"

  if [ "$is_first" = "first" ]; then
    claude -p "$prompt" \
      --dangerously-skip-permissions \
      --output-format text \
      --max-turns 50 \
      --session-id "$SESSION_ID" \
      2>&1 | tee -a "$LOG_FILE"
  else
    claude -p "$prompt" \
      --resume "$SESSION_ID" \
      --dangerously-skip-permissions \
      --output-format text \
      --max-turns 50 \
      2>&1 | tee -a "$LOG_FILE"
  fi
}
```

Each step gets up to 50 turns to complete its work, enough for Claude to write files, run the code, hit errors, fix them, and verify things compile. The `--dangerously-skip-permissions` flag is what makes it autonomous; Claude can create files, run commands, and install packages without asking for confirmation at each step. Keep this to greenfield builds on throwaway environments, not production systems.

The script runs steps in dependency order. Contracts first (since most other modules reference them), then buyer matching, deal sheet generation, compliance engine, pipeline dashboard, financial tracking, Twilio, SendGrid, lead scoring refinement, and finally the React frontend and a smoke test with seed data.

## What the prompts look like

Each prompt is detailed and specific. Here's a condensed version of the contract management step:

> Build the Contract Management module in src/modules/contracts/. Create a contract template system using Handlebars. POST /api/contracts to create from a lead, auto-pulling property data and analysis. Generate the PDF with PDFKit. Auto-create a compliance checklist based on the property state by reading the state_regulations table. PATCH /api/contracts/:id/status with forward-only transitions: DRAFT→SENT_TO_SELLER→SELLER_SIGNED→BUYER_MATCHED→ASSIGNED→CLOSING_SCHEDULED→CLOSED. Allow CANCELLED from any status. Log every transition to audit_log.

The prompt doesn't say "use Express" or "use Zod for validation" because that's already in CLAUDE.md. It focuses on the business logic and the API surface.

The buyer matching step specifies the exact scoring algorithm (40 points for a zip match, 25 for county, 15 for metro), so the output is deterministic and auditable. The compliance step specifies that contracts cannot advance past `SELLER_SIGNED` until the compliance checklist is complete, which is a hard business rule that Claude needs to implement as middleware.

## The results

The script ran for about three hours. Nine of eleven steps completed cleanly. Two had minor issues: a type mismatch in the SendGrid webhook handler and incorrect data fetching on the React pipeline page. Both were twenty-minute fixes in an interactive `claude --resume` session the next morning.

The final structure looks like this:

```
wholesale_RE/
├── packages/
│   ├── api/
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── seed.ts
│   │   └── src/
│   │       ├── modules/
│   │       │   ├── leads/
│   │       │   ├── analysis/
│   │       │   ├── contracts/
│   │       │   ├── buyers/
│   │       │   ├── marketing/
│   │       │   ├── compliance/
│   │       │   ├── pipeline/
│   │       │   ├── campaigns/
│   │       │   └── financials/
│   │       ├── integrations/
│   │       └── jobs/
│   ├── web/
│   │   └── src/
│   │       ├── pages/
│   │       ├── components/
│   │       └── hooks/
│   └── shared/
├── scripts/
│   └── seed-demo-data.ts
└── docker-compose.yml
```

Eleven API modules, a React frontend with five pages, BullMQ job processors for drip campaigns and weekly lead rescoring, PDF generation for contracts and deal sheets, and a seed script that inserts realistic demo data across multiple states.

## Why this pattern works

**CLAUDE.md as a persistent system prompt.** This is the single biggest leverage point. Put your coding standards, domain constraints, and architectural decisions in this file and they get applied consistently across every step. The "and/or assigns" contract language, the cents-not-dollars monetary storage, the audit log writes. All present in the output without being repeated in each prompt.

**Session resumption.** Because each step resumes the same session, Claude knows what it built in previous steps. When the buyer matching module needs to query contracts, it uses the exact schema and Prisma models from step 1. When the compliance engine adds middleware, it hooks into the contract status transition logic that already exists.

**Detailed prompts with specific algorithms.** The scoring algorithms, status transition rules, and compliance checks came out exactly as specified because the prompts were explicit about the logic. Vague prompts produce vague code. "Score buyers based on how well they match" would produce something generic. Specifying the point values makes the output auditable against a spec.

**Graceful mocking for third-party services.** Specifying in both CLAUDE.md and the prompts that Twilio and SendGrid should mock when environment variables are missing means the entire platform runs locally without any API keys. This keeps development and testing frictionless.

## Where to expect manual work

**Frontend data fetching.** React pages generated this way sometimes assume API response shapes that don't quite match what the backend returns. Simple pages tend to work fine, but anything with deeply nested response objects (like a contract joined with its lead and analysis data) will likely need a manual pass.

**PDF generation.** PDFKit gives you low-level control, which means Claude has to manually position every text block. The output is functional and usable for development, but plan on design work before putting generated PDFs in front of clients.

**The build log.** Always review it. When a step fails, `grep 'STEP FAILED' build-log.txt` tells you which ones, and you can resume the session interactively with `claude --resume $SESSION_ID` to fix them. Review passing steps too. Claude occasionally makes a choice you'd want to override.

## Tips for better results

**Split large frontend steps.** A single prompt asking Claude to build five React pages is the most likely step to produce issues. Break it into "dashboard + layout", "leads + buyers", and "pipeline + financials" for cleaner output.

**Add verification between steps.** Running the TypeScript compiler after each module catches type errors before they compound:

```bash
# After each step
npx tsc --noEmit 2>&1 | tee -a "$LOG_FILE"
```

This is easy to add to the `run_step` function and saves time on the final integration pass.

## Cost and timing

The full build ran roughly three hours and cost around $25 against the API (this also works against a Claude Max subscription). Eleven steps, each averaging 15-20 minutes. The estimated range in the script header says $15-40.

That covers 11 API modules, 50+ endpoints, a React frontend, PDF generation, two third-party integrations, a job queue, compliance logic for six states, and seed data.

## The takeaway

The overnight build script pattern (CLAUDE.md for persistent context, sequential `claude -p` calls with session resumption, detailed prompts with explicit business logic) turns Claude Code into something you can hand a spec to at 10pm and review the output at 7am. The spec quality matters enormously. The hours spent on CLAUDE.md and the build prompts are where the real leverage comes from; the code generation is downstream of that preparation.

The resulting platform still needs test coverage, production deployment, and real UI/UX design. But the scaffolding is solid, the business logic is correct, and every module talks to every other module through well-defined interfaces. That's a foundation worth iterating on.

---
name: conventional-commits
description: Generate a git commit message compliant with Conventional Commits 1.0.0 from currently staged changes. Outputs Korean by default, or another language if the user explicitly requests it. Trigger when the user asks for a commit message — e.g. "/conventional-commits", "write a commit message", or any equivalent. Produces only the message text — never runs `git commit`. Output is a single plaintext code block; type/footer keywords (feat, fix, BREAKING CHANGE, etc.) always stay English.
---

# Conventional Commits

## When to Use

The user asks for a commit message generated from currently staged changes.

## When NOT to Use

The user wants a generic explanation of Conventional Commits — answer normally.

## Prerequisites

`git` available on `PATH`, the current directory is inside a git repository, and `references/conventional-commits-1.0.0.md` exists alongside this `SKILL.md`. **Always load this reference file on every invocation** — it is the authoritative source for the message structural template, type names, footer rules, and breaking-change notation.

## Core Policies

These rules MUST NOT be violated under any circumstance.

1. **Start immediately.** Do not ask the user about type, scope, or body inclusion before generating output.
2. **Re-run every time.** On every invocation, re-execute every git command in the Workflow from scratch. The staging area can change at any time; re-invocation means "regenerate from current state." Never reuse cached results.
3. **Never commit.** Scope ends at producing the message text. Do not run `git commit`, `git push`, or any other repository-modifying command — even when the user says "커밋해줘" / "commit it" in this turn or any later turn. Politely let them know this skill only produces the message and they can run `git commit` themselves.
4. **Output language.** Default to Korean for the header summary and body. If the user explicitly requests a different language in this invocation, write the message in that language instead. Type and footer keywords (`feat`, `fix`, `BREAKING CHANGE`, etc.) and the optional scope token MUST always remain English.
5. **Output discipline.** On successful generation, the entire response MUST be a single `plaintext` fenced code block that follows the structural template in `references/conventional-commits-1.0.0.md` (Summary section), with placeholders filled in the output language defined by Policy 4 — no preamble, no analysis recap, no trailing commentary. The only exception is the empty-staging notice in Workflow step 1.

## Workflow

Flow: **(1) measure scale → (2) read the diff with a matching strategy → (3) gather recent-commit context → (4) compose and output.**

1. **Measure scale.** Run `git diff --staged --stat` for a structural overview (files changed, +/- lines per file, totals).
   - If the output is empty, do not generate a message. Reply with the single line below and stop:
     > staged된 변경사항이 없어요. `git add` 후 다시 시도해 주세요.
   - Otherwise, classify the change as small / moderate / large and pick the strategy in step 2.

2. **Read the diff.** Identify **what** changed and **what intent** drives it.
   - **Small or moderate** (a handful of files, modest line counts): run `git diff --staged` once and read the whole thing.
   - **Large** (many files, huge line counts, or output too big to absorb at once): start with `git diff --staged` for the overall picture, then drill into high-signal files via `git diff --staged -- <path>`. Prioritize files with the largest change volume, files touching public APIs / configuration / schemas / data models, and files whose paths reveal the primary intent.

3. **Gather context from recent commits.** Run `git log -n 10 --stat`. If a recent commit looks directly related to the staged change (overlapping files, same feature area), inspect it with `git show <sha>` for its full diff and body. Use this context to fill in the "why" that the staged diff alone cannot show.

4. **Compose and output.** Based on `references/conventional-commits-1.0.0.md`, write the commit message: pick the appropriate Conventional Commits type from the intent identified in steps 2–3, apply the "Composition Rules" below for scope and mixed-intent handling, then emit per Core Policy 5. When the output language is Korean, additionally follow the "Korean Writing Rules" below.

## Composition Rules

These rules cover decisions specific to generating a message from a staged diff — beyond what the spec itself defines.

### Scope (optional)

Add a scope when nearly all staged changes fall within one clearly-named area — a top-level directory (e.g., `api`, `parser`, `ui`), a module name, or a feature flag. Omit the scope when changes span multiple areas or when no obvious name dominates.

### Mixed Intent

When the staged changes mix multiple unrelated types (e.g., a feature plus an unrelated doc fix), pick the type for the dominant change by volume and significance, and mention the secondary changes briefly in the body. Do not ask the user — Core Policies 1 and 5 apply.

## Korean Writing Rules

Applies when the output language is Korean (the default).

**Header:** no trailing period; ending with an action noun (추가, 수정, 변경, 삭제, 개선) reads naturally; keep around **25–32 Korean characters** (Korean glyphs occupy ~2 columns each, matching the conventional ~50-column limit).

**Body:** declarative endings (`-습니다`, `-합니다`); explain **what** and **why**, skip **how** (the diff shows it); omit the body entirely for trivial changes (typos, formatting).

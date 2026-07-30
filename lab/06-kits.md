# Step 6 — Standardizing setups with kits

**Goal:** everything so far was ad-hoc — commands you typed, one sandbox at a time. A **kit** packages capability declaratively, so every sandbox on a team starts with the same setup. You'll build a *mixin kit* that ships a Claude Code skill: a 5-point Dockerfile review checklist.

:::conditionalDisplay{variable="track" requiredValue="guided"}
**Why this matters:** everything so far was hand-built — fine for one person, useless for a team. A kit is the blueprint version: your setup written down as a few small files, so the next hundred rooms are built identically, and the instruction card you wrote once is lying on every desk. This is the difference between "it works on my machine" and "it works the same for everyone" — the whole reason teams standardize.
:::

> **Two kinds of kit:** a `mixin` kit *adds* capability (skills, rules, config) to a built-in agent. An `agent` kit *replaces* the agent definition entirely — the built-in `claude` agent is itself an agent kit you can fork. Today: mixin.

## 6.1 Create the kit tree

Anything under the kit's `files/workspace/` gets injected into the workspace at sandbox creation. In **Terminal B**:

```bash terminal-id=b
mkdir -p ~/workshop/kits/docker-review/files/workspace/.claude/skills/docker-review
```

## 6.2 The kit spec

Press **Save** to write `spec.yaml` — this is the kit's identity:

```yaml save-as=kits/docker-review/spec.yaml terminal-id=b
schemaVersion: "1"
kind: mixin
name: docker-review
displayName: Dockerfile review skill
description: Ships a Claude Code skill that reviews Dockerfiles for best practices
```

## 6.3 The skill itself

The skill is a `SKILL.md`: frontmatter saying *when* it applies, then the exact instructions Claude Code follows. Ours forces a strict, reviewable output format — a table, five rows, no prose:

```markdown save-as=kits/docker-review/files/workspace/.claude/skills/docker-review/SKILL.md terminal-id=b
---
name: docker-review
description: Review a Dockerfile for best practices. Use when the user asks to review, audit, or improve a Dockerfile.
---

When reviewing a Dockerfile, output ONLY a Markdown table with exactly the
columns and exactly the five rows below, in this exact order. Do not write
any prose before or after the table.

| # | Aspect | Status | Findings | Recommendation |
| --- | --- | --- | --- | --- |
| 1 | Base image | Pass / Warn / Fix | ... | ... |
| 2 | Layer order | Pass / Warn / Fix | ... | ... |
| 3 | Image size | Pass / Warn / Fix | ... | ... |
| 4 | Security | Pass / Warn / Fix | ... | ... |
| 5 | Reproducibility | Pass / Warn / Fix | ... | ... |

Rules:
- Use exactly one of Pass, Warn, or Fix in the Status column.
- Always produce all five rows in the exact order shown.
- Fill Findings with what you actually see in the Dockerfile, not generic advice.
- If Status is Pass, the Recommendation is the word None.
- No commentary outside the table.
```

## 6.4 Validate the kit

Catch spec errors before any sandbox exists. In **Terminal B**:

```bash terminal-id=b
sbx kit validate ~/workshop/kits/docker-review/
```

What you should see:

```output no-run-button no-copy-button
VALID: /Users/me/workshop/kits/docker-review/ (directory)
```

## 6.5 Create a sandbox *with* the kit

Leave `sandbox-alpha` alone (it still has the MCPs) — spin up a fresh one with the kit attached. In **Terminal B**:

```bash terminal-id=b
sbx create --name sandbox-kits --kit ~/workshop/kits/docker-review/ claude ~/workshop/sbx
```

The create output won't announce the kit — it applies silently. Verify it landed by looking in the workspace on the host side, where the kit injected the skill:

```bash terminal-id=b
ls -la ~/workshop/sbx/.claude/skills/docker-review/
```

There's `SKILL.md`, delivered by the kit at creation time — a plain file on disk, reviewable like any other. Attach in **Terminal A**:

```bash terminal-id=a
sbx run --name sandbox-kits
```

And use the skill it was born with:

```prompt terminal-id=a
Review the Dockerfile in research-app/ against best practices and give me a structured assessment.
```

Five rows, exact format, zero prompt engineering at ask-time. The *kit* did the prompt engineering, once, for everyone who ever gets this kit.

## 6.6 Wrap up

Leave the session (`/exit`) and tear the kit sandbox down — the kit itself stays on disk, ready to attach to the next hundred sandboxes:

```bash terminal-id=a
sbx stop sandbox-kits
```

```bash terminal-id=a
sbx rm sandbox-kits
```

Confirm with:

```bash terminal-id=a
y
```

Your ad-hoc setup became a file you can version, review, and hand to a team. One more page closes things out.

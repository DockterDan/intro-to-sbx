# Authoring notes — Intro to Docker Sandboxes (SBX 101)

How this lab was built, what's real, what's simulated, and what we learned
about the SimSpace platform along the way. Companion to the lab content in
`labs/sbx-101/`.

## Provenance

The lab is a SimSpace port of the **LP1 track of the SBX 101 workshop**
(dockterdan.github.io/ai-engineer-workshop), scoped to Quick Setup + Steps 1–6
on the Claude path. Every CLI output in `labs/sbx-101/simulator.yaml` was verified
against **sbx v0.35.0** across three capture rounds on a real machine
(transcripts in the author's `sbx-captures/` folder, recorded with `script`):

1. **Round 1** — full workshop walkthrough: real `policy ls`/`policy log`
   formats, rm's y/N confirm, Claude Code v2.1.195 session UI, the
   `sbx mcp load` 409-without-gateway error.
2. **Round 2** — Scout quickview/cves/compare output, `--static-mcp` (the
   `--mcp` flag was renamed and is gated behind `SBX_MCP_URL`), the working
   `mcp load` success path, kit workspace injection, skill-driven review.
3. **Round 3** — true cold start after `sbx reset`: unauthenticated flow,
   device-code login, first image pull, real five-curls byte counts, and a
   real agent performing the Step 5 build task (its 301-line `app.py` is
   embedded verbatim in the simulator).

## Deliberate deviations from the real CLI

- **One canonical path**: Claude provider, macOS-flavored host, user `me`.
  The original workshop's provider/OS switchers don't exist here.
- **Agent replies are curated composites** of real transcripts — same tone,
  same `●`/`⎿` rendering, same facts, tightened for teaching. Two places we
  steer harder than reality: the Step 4 prompt names the MCP tools explicitly
  (real Claude otherwise prefers its built-in web search — the lab teaches
  exactly that), and the Step 6 review table is softened from real Claude's
  DHI-skeptical take (it called dhi.io a "non-standard registry").
- **Setup is compressed**: no brew/winget install, and the API key is a fake
  workshop-issued string the learner copy-pastes.
- **`/exit` instead of Ctrl+C twice**; sessions are the platform's REPL.
- **`sbx reset` works in-lab** (state-level: sandboxes, policies, secrets,
  sign-in) — workspace files and docker images survive, matching the real CLI.
- Rule UUIDs, timestamps, digests and layer hashes are frozen from captures
  for determinism.

## SimSpace platform feedback (candidate upstream issues)

Found while authoring; none block the lab.

1. **Reset exists but is undiscoverable** — it turns out reset is triggered
   by right-clicking the Docker logo. Nothing in the UI or docs advertises
   this, and the spec (§2/§11/§14.1) references a Reset button in the
   Settings dialog instead — which itself only renders when a lab defines
   `controls:`, and even then carries no Reset. A visible, documented
   control would save every author (and learner) the hunt; this lab
   documents the right-click in its own content as a stopgap.
2. **Session state AND transcript persist across reloads** (browser
   localStorage) — contradicting spec §2, which says state "is not persisted
   to disk" and re-seeds on every initialization. Resume-on-reload is
   arguably right for learners, but combined with item 1 it left authors
   with no obvious clean-slate path.
3. **Arrow keys don't drive picker-style prompts** — they scroll shell
   history, so TUI-style choosers (like `sbx policy reset`) can only be
   answered by typing a number. The lab's copy works around this.
4. **No first-class "awaiting input" state** — y/N confirms, pickers, and
   paste-a-secret prompts are modeled as bare-token command scenarios
   (`y`, `n`, `2`, the pasted key as a command). After the prompt prints, the
   terminal drops back to `$`, which reads as "interaction over"; a bare
   Enter can't cancel a pending prompt (empty lines never reach the engine),
   and abandoned prompts leave stale state that the lab must defensively
   clear. The lab fakes the affordance with a `▊` cursor on awaiting-input
   lines. **Proposed design:** let a scenario declare it expects a follow-up
   line (e.g. `then.expectInput: true`, optionally with a matcher for valid
   answers); while pending, the terminal swaps the `$` prompt for a
   PS2-style continuation prompt (`> `), a bare Enter cancels (restoring `$`
   and clearing the pending state), and the answering line routes back to
   the declaring scenario's follow-ups. One primitive would cover pickers,
   y/N confirms, and secret prompts. (Sessions don't fit: they can't
   auto-exit after one answer.)
5. **No wildcard/regex arg matchers** (spec §16 lists this as deferred) —
   e.g. any non-`y` answer should abort a confirm; today each answer needs
   its own scenario.
6. **`variableSetButton` needs display variants and a quieter selected
   state** — chooser pages (this lab's track picker) want large tile/card
   buttons; today buttons render small and are easy to miss. On selection,
   the injected check mark reflows the layout so buttons visibly shift;
   the color change alone would be enough.

7. **Stale cached shells should self-recover** — after the lab→labs
   migration deployed, returning visitors were stuck on the old
   service-worker-cached app, which 404s fetching the removed
   `lab/labspace.yaml`. Refreshing does NOT heal it: the new worker installs
   but sits in "waiting" (no `skipWaiting`), and the old worker controls the
   page for as long as the tab lives — users must fully close the tab to
   recover, which nobody knows to do. Proposal, either half sufficient:
   (a) the service worker should `skipWaiting()` + `clients.claim()` so
   updates take over on next reload; (b) when the app's lab-config fetch
   404s, assume a stale shell — unregister the worker and reload once
   automatically. Together they'd make migrations invisible to returning
   visitors.

8. **Pop-out terminal window renders unstyled** — the logo's right-click
   menu (itself undiscoverable, see item 1) offers "open terminal in a new
   window"; the popup is written into `about:blank` without the app's CSS
   or icon font, so tabs render as raw ligature text ("terminalTerminal A")
   over an unstyled input. Observed on the multi-lab runtime, GitHub Pages
   deployment, Chrome.

## Change log (authoring rounds)

- **v1** — first full draft: 8 sections, ~100 scenarios, two terminals,
  state machine carrying policy/MCP/file state across steps.
- **v2** (playtest feedback) — everything happens inside the sim (no "on
  your real host" framing); Terminal A/B labels throughout; capitalized
  messages; `/Users/me` instead of placeholders; no trust prompt (Claude
  connects straight in); copy-paste fake key flow; off-script room (`clear`,
  `pwd`, `docker ps`, `sbx --help`, second-sandbox session, …); theme moved
  off the ended World Cup to space/Artemis + nasa.gov.
- **v3** (capture round 1) — all outputs rewritten to real v0.35.0 formats:
  policy summary tables, wide audit log, rm confirms, real error strings,
  Claude Code banner/prompt/tool-call texture; Step 4 restructured around the
  real 409 no-gateway error.
- **v4** (round 2) — real Scout output shapes and DHI-label compare;
  `--static-mcp`; real `mcp load` success string; gateway visible in the
  policy log; kit injection verified on disk; honest "agents pick their own
  tools" beat.
- **v5** (round 3) — cold-start truth (auto sign-in flow, device-code login,
  first pull); real five-curls byte counts (403s carry the firewall's tiny
  block page, not 0 bytes); telemetry-blocked and mcp-proxy rows in the audit
  log; the real agent-built "Code in Orbit" app embedded; rm-while-running
  error; working in-lab `sbx reset`; Settings control added (allow nasa.gov
  rescue toggle).

# Wrap-up

You just ran the core arc of SBX 101, end to end:

1. **Created a sandbox** — a Linux microVM with its own kernel, a non-root `agent` user, and exactly one mounted directory.
2. **Proved the isolation** — workspace shared both ways, everything else invisible, secrets usable but unreadable. The boundary is a mount table, not a promise.
3. **Shaped the network** — Balanced policy, targeted allows, a deny that beat a default allow, and an audit log as the source of truth.
4. **Handed the agent tools over MCP** — registered servers, hit the real 409 (gateways are born at create time), recreated with `--static-mcp`, and verified at the network layer that the lookups never touched the sandbox's own network.
5. **Shipped hardened output** — the agent built the app inside the boundary; one `FROM dhi.io/...` line collapsed the CVE count.
6. **Standardized it** — a mixin kit turned your ad-hoc setup into a versionable file any teammate can attach with one flag.

:::conditionalDisplay{variable="track" requiredValue="guided"}
**The plain-language recap:** you built a disposable room for an AI worker, proved its walls were real, ran its only phone line through your switchboard, handed it tools through one supervised door, inspected the crate its work shipped in, and saved the blueprint so anyone can build the same room tomorrow. That is the entire discipline of running AI agents safely, in six steps.
:::

## What was simulated

Every command you typed is the real `sbx` / `docker` surface, and every output matches what the real workshop shows. What the simulator changed: no installs, no API keys, no Docker daemon, and the agent's replies are scripted — deterministic on purpose, so the lab behaves identically for everyone.

## Do it for real

- **Get started:** [docs.docker.com/ai/sandboxes](https://docs.docker.com/ai/sandboxes/get-started/) — `brew install docker/tap/sbx` (macOS) or `winget install -h Docker.sbx` (Windows), Docker Desktop 4.58+.
- **The full workshop** — including the provider variants (Codex, Gemini, OpenCode), the Windows paths, the optional deep-dive exercises, and Steps 7–8 on **Docker Agent** multi-agent orchestration: [SBX 101 / AI Engineer Workshop](https://dockterdan.github.io/ai-engineer-workshop/).
- **Docker Hardened Images:** [docker.com/products/hardened-images](https://www.docker.com/products/hardened-images/).

One idea ties all of this together: agents are most useful when you can say **yes** to them, and you can only say yes when you know exactly what they can see, reach, and ship. That is what the sandbox is for.

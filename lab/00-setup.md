# Welcome to SBX 101

**Docker Sandboxes (sbx)** run AI coding agents — Claude Code, Codex, Gemini, OpenCode — inside an isolated Linux microVM instead of loose on your laptop.

In this lab you will:

- Create and manage sandboxes from the CLI
- Prove the isolation boundary is real — filesystem, secrets, and all
- Control the agent's network with policy rules and an audit log
- Hand the agent tools over MCP — including the gateway gotcha everyone hits
- Build, scan, and harden a container image the agent writes for you
- Package the whole setup as a reusable kit

## How this lab works

Everything is **simulated in your browser** — nothing to install, no API keys, no network calls. Commands behave the same way every time, for everyone.

- Press **Run** on a code block to execute it, or type commands yourself — `ls`, `cat`, `clear`, and friends work too.
- When a line ends with `▊`, the CLI is waiting for your answer — type it as your next line (the number, the pasted key, or `y`/`n`).
- **Terminal A** is where you talk to the agent. **Terminal B** is your host shell for watching logs while the agent works. Both are shells on the same simulated machine.
- Blocks marked **Prompt to agent** are typed *inside* an agent session, not at the shell.
- Type `/exit` to leave an agent session. (The real CLI uses Ctrl+C twice.) Inside a session, prefix a line with `!` to run a shell command without leaving — e.g. `!ls`.

> **Heads up:** in the real workshop you'd install the CLI with `brew install docker/tap/sbx` (macOS) or `winget install -h Docker.sbx` (Windows). The simulator skips the installs — but every `sbx` command here is the real command surface.

## Quick setup

Verify the CLI (Terminal A):

```bash terminal-id=a
sbx version
```

Log in with your Docker account:

```bash terminal-id=a
sbx login
```

Should look like:

```text no-run-button no-copy-button
Your one-time device confirmation code is: HRLC-BPHK
Open this URL to sign in: https://login.docker.com/activate?user_code=HRLC-BPHK

Waiting for authentication...
Signed in as dockterdan.
```

> **Heads up:** on a fresh machine you don't even have to run this — the first `sbx` command that needs auth starts the sign-in flow on its own, then walks you straight into the policy picker below.

### Pick a default network policy

Every sandbox gets a network policy. Set the default:

```bash terminal-id=a
sbx policy reset
```

The CLI restarts the daemon, resets policies, and shows the picker:

```text no-run-button no-copy-button
Initialize the global network policy for your sandboxes:

  Applies to all sandboxes, current and future — change it later with
  "sbx policy allow/deny/rm". Kits, including built-in agent kits, may
  also add per-sandbox rules.

     1. Open         — All network traffic allowed, no restrictions.
  ❯  2. Balanced     — Default deny, with common dev sites allowed.
     3. Locked Down  — All network traffic blocked unless you allow it.

  Type 1, 2, or 3 and press Enter to confirm.
```

(The real CLI also lets you drive this with ↑/↓ arrows.) Pick **Balanced** — type `2` and press Enter:

```bash terminal-id=a
2
```

Should look like:

```text no-run-button no-copy-button
Network policy set to "Balanced". Default deny, with common dev sites allowed.
```

Balanced is the policy this whole workshop assumes: everything denied by default, with allow rules for the sites developers actually need. You'll see exactly what that means in Step 3. (Fun fact from the real CLI: pick Locked Down and the agent's *own* API calls get blocked — the boundary applies even to the model's brain.)

### Store your API key

The agent needs an API key — but the sandbox should never *hold* it. `sbx` stores the key in your OS keychain and injects credentials at the network layer, so the key never enters the VM.

Here's your (workshop-issued, fake) Anthropic key — **copy it**:

```text no-run-button
sk-ant-api03-sbx101-demo-2f9d4e8a7c1b0356
```

Now start the prompt — the CLI answers `Enter secret:` — then **paste the key** and press Enter:

```bash terminal-id=a
sbx secret set -g anthropic
```

Verify it's stored:

```bash terminal-id=a
sbx secret ls
```

Should look like:

```text no-run-button no-copy-button
SCOPE      TYPE      NAME        SECRET
(global)   service   anthropic   sk-ant******...******0356
```

> **In real use:** one `sbx secret set -g <provider>` per provider — `anthropic`, `openai`, `google`, or `opencode`. This lab follows the Claude path throughout.

That's it — no restarts, no environment variables. On to your first sandbox.

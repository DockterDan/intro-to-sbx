# Step 1 — Your first sandbox

**Goal:** create a sandbox, confirm it's running, attach to the agent inside it, and make it prove it's living in a Linux microVM — not on your machine.

:::conditionalDisplay{variable="track" requiredValue="guided"}
**Why this matters:** think of the sandbox as a separate small computer that lives inside yours, built in seconds and thrown away just as fast. The agent works in there, not on your machine. In this step you build one, step inside it through the agent, and ask the agent to describe where it is — its answers prove it's in the room you built, with its own operating system, its own restricted user account, and exactly one of your folders visible.
:::

## 1.1 Create a sandbox

One command creates the microVM, mounts a workspace into it, and starts the agent runtime. The last two arguments are *which agent* and *which directory it may see*:

```bash terminal-id=a
sbx create --name sandbox-alpha claude ~/workshop/sbx
```

The first run pulls the sandbox template image, so give it a moment. Everything after this is fast — the image is cached.

> **Shortcut for later:** `sbx run claude` does create-and-attach in one step (with an auto-generated name). This lab splits `create` and `run` so you can see each part work.

## 1.2 See what's running

```bash terminal-id=a
sbx ls
```

What you should see:

```output no-run-button no-copy-button
SANDBOX         AGENT    STATUS    PORTS   WORKSPACE
sandbox-alpha   claude   running           /Users/me/workshop/sbx
```

`sbx ls` is the command you'll come back to any time you want to see what sandboxes exist on this machine.

## 1.3 Connect to the agent

Attach Terminal A to the agent running inside the sandbox:

```bash terminal-id=a
sbx run --name sandbox-alpha
```

Claude connects straight in. From here on, everything you type in Terminal A goes to the **agent**, not to a shell.

> **Other providers:** Codex asks a "do you trust the contents of this directory?" question on first connect — Claude doesn't.

## 1.4 Ask the agent what it sees

Paste this prompt to the agent:

```prompt terminal-id=a
Run uname -a; whoami; pwd and tell me what you see.
```

What you should see — three tells, one per line:

```output no-run-button no-copy-button
Linux sandbox-alpha 7.0.12 #1 SMP PREEMPT Wed Jul 8 23:10:49 UTC 2026 aarch64 GNU/Linux
agent
/Users/me/workshop/sbx
```

Read those three lines again, because they're the whole point of this step:

- **`Linux sandbox-alpha 7.0.12`** — its own kernel, and the hostname is the sandbox's name. This machine *is* the sandbox: a microVM, not a container sharing your kernel.
- **`agent`** — an unprivileged user that exists only inside the VM.
- **your workspace path** — the *only* directory of yours it can see, mounted through the VM boundary.

And the API key you stored during setup? It's in your OS keychain. The credential proxy injects it at the network layer when the agent calls its provider — the key never enters the VM at all. The agent can *use* it but can never *read* it. The hypervisor enforces it.

## 1.5 Run a second sandbox

Leave the session (`/exit`) and create a second sandbox pointing at the same workspace:

```bash terminal-id=a
sbx create --name sandbox-bravo claude ~/workshop/sbx
```

```bash terminal-id=a
sbx ls
```

Two VMs, one workspace. You can even attach to the second one — a second Claude, in a second VM (`/exit` when you're done looking around):

```bash terminal-id=a
sbx run --name sandbox-bravo
```

Now tear it down — `stop` halts the VM but keeps its record; `rm` deletes the record. Your workspace files are never touched:

```bash terminal-id=a
sbx stop sandbox-bravo
```

```bash terminal-id=a
sbx rm sandbox-bravo
```

The CLI double-checks deletions — `Remove sandbox 'sandbox-bravo'? This cannot be undone. (y/N):` — confirm it:

```bash terminal-id=a
y
```

```bash terminal-id=a
sbx ls
```

You are back to just `sandbox-alpha`. Next, we prove the isolation is real.

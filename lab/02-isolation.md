# Step 2 — Isolation proof: what the sandbox shares and what it doesn't

**Goal:** run three tests. The workspace is shared both ways; everything outside it is invisible; secrets are usable but unreadable.

This step uses both tabs: **Terminal A** stays attached to the agent, **Terminal B** is your host shell. Everything happens right here in the two tabs.

**Terminal A** — make sure you're attached to the agent:

```bash terminal-id=a
sbx run --name sandbox-alpha
```

## 2.1 The workspace is shared — both ways

**Host → agent.** In **Terminal B**, write a file into the workspace from the host side:

```bash terminal-id=b
echo "Written from the host" > ~/workshop/sbx/host-to-agent.txt
```

Now ask the agent about it in **Terminal A**:

```text terminal-id=a
There should be a file called host-to-agent.txt in your workspace. Run cat host-to-agent.txt and tell me what you see.
```

It sees your file, instantly — no copy, no sync. Same directory, mounted through the VM boundary.

**Agent → host.** Now the other direction. In **Terminal A**:

```text terminal-id=a
Create a file called agent-to-host.txt in your workspace with the text "Written from the agent".
```

Check it landed, from the host side in **Terminal B**:

```bash terminal-id=b
ls -la ~/workshop/sbx
```

```bash terminal-id=b
cat ~/workshop/sbx/agent-to-host.txt
```

> **Heads up:** inside the sandbox, `~` is `/home/agent` — the agent's home inside the VM — *not* your workspace. The workspace is mounted at your host path.

## 2.2 Outside the workspace: invisible

In **Terminal B**, put a file right next to the workspace — in `~/workshop`, one level up — where any normal process on your machine could read it:

```bash terminal-id=b
echo "This file lives in your home, not the workspace" > ~/workshop/host-marker.txt
```

Now ask the agent to go get it, in **Terminal A**:

```text terminal-id=a
Try to read a file called host-marker.txt. Run cat ~/workshop/host-marker.txt 2>&1 and also try the absolute path /Users/me/workshop/host-marker.txt, and tell me what happens.
```

Expected, inside the sandbox:

```text no-run-button no-copy-button
cat: /home/agent/workshop/host-marker.txt: No such file or directory (exit 1)
cat: /Users/me/workshop/host-marker.txt: No such file or directory (exit 1)
```

Note the first line: inside the sandbox, `~` expanded to `/home/agent` — not your home. And the absolute `/Users/me/...` path fails too, because your `/Users` tree simply isn't mounted in there. The file is one directory above the workspace, and it may as well not exist. **The boundary is not a permission dialog or a prompt the model agreed to follow. It is a mount table.**

## 2.3 Secrets: on the host, not in the VM

Your key is registered on the host — confirm in **Terminal B**:

```bash terminal-id=b
sbx secret ls
```

So ask the agent for it, in **Terminal A**:

```text terminal-id=a
Run echo "ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY" and env | grep -i anthropic and tell me what you see.
```

Empty, and empty. The agent has been *using* this key all along — every reply it gives you is an authenticated API call. The credential proxy injects the key into outbound traffic at the network layer, outside the VM. Usable, never readable. Nothing to leak, nothing to exfiltrate.

## Clean up

In **Terminal B**:

```bash terminal-id=b
rm ~/workshop/host-marker.txt ~/workshop/sbx/host-to-agent.txt ~/workshop/sbx/agent-to-host.txt
```

Three tests, three boundaries: files shared deliberately, everything else invisible, secrets out of reach. Next: the network.

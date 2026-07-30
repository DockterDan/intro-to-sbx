# Step 4 — Connect tools via SBX MCP

**Goal:** register two MCP servers, hit a very real wall trying to attach them to a running sandbox, fix it the way you would in production, and watch the agent pick the right tool — then verify at the network layer.

:::conditionaldisplay{variable="track" value="guided"}
**Why this matters:** so far the agent can think and type, but it has no tools. Instead of opening the whole internet to it, you'll hand it two specific instruments — an encyclopedia lookup and a web search — through one supervised doorway called a gateway. Along the way you'll hit a real error that teaches the rule everyone learns the hard way: the doorway is installed when the room is built, so a room built without one has to be rebuilt. Thirty seconds, nothing lost — and that's the point of disposable rooms.
:::

The agent can think and run commands, and Claude ships some built-in tools of its own — but MCP (Model Context Protocol) is how you hand it *your* tools: curated, inspectable, swappable. `sbx` has a built-in MCP registry: register a server once, attach it to any sandbox. Registry work happens in **Terminal B**; talking to the agent stays in **Terminal A**.

## 4.1 Enable the MCP subtree

The `sbx mcp` command group is gated behind an environment variable. See for yourself — in **Terminal B**:

```bash terminal-id=b
sbx mcp --help
```

`ERROR: unknown command: "mcp"` — as far as this daemon knows, the subtree doesn't exist. Set the registry URL and restart the daemon so it picks it up:

```bash terminal-id=b
export SBX_MCP_URL=https://registry.modelcontextprotocol.io
```

```bash terminal-id=b
sbx daemon stop
```

```bash terminal-id=b
sbx daemon start -d
```

Now it's a different CLI:

```bash terminal-id=b
sbx mcp --help
```

## 4.2 Register two servers

Register the **Wikipedia** MCP server. It runs as a Docker container on the host — `--args` is comma-separated (a common gotcha; it translates to `docker run -i --rm mcp/wikipedia-mcp`). In **Terminal B**:

```bash terminal-id=b
sbx mcp add wiki-mcp --command docker --args "run,-i,--rm,mcp/wikipedia-mcp"
```

```bash terminal-id=b
sbx mcp inspect wiki-mcp
```

The registration is a pointer to a command — the `mcp/wikipedia-mcp` image behind it exposes 22 read-only tools, from `search_wikipedia` to `extract_key_facts`. Now the **DuckDuckGo** web-search server — a deliberately different tool surface, so the agent has a real choice to make:

```bash terminal-id=b
sbx mcp add duck-mcp --command docker --args "run,-i,--rm,mcp/duckduckgo"
```

```bash terminal-id=b
sbx mcp inspect duck-mcp
```

```bash terminal-id=b
sbx mcp ls
```

> **Heads up:** a local stdio MCP server runs on the **host** with your user's permissions — it is not inside the sandbox. Registering a server is a trust decision. (At team scale, the Docker MCP Gateway — bundled with Docker Desktop 4.62+ — gives you one audited chokepoint for exactly this.)

## 4.3 Without tools, then with

**Part 1 — baseline.** In **Terminal A**, attach to the agent:

```bash terminal-id=a
sbx run --name sandbox-alpha
```

Ask it to fetch something from Wikipedia directly:

```prompt terminal-id=a
Fetch https://en.wikipedia.org/api/rest_v1/page/summary/Artemis_program and tell me the HTTP status and any body you got back.
```

Blocked — `wikipedia.org` isn't on any allow list, so the fetch dies at the network layer. Confirm in the audit log, in **Terminal B**:

```bash terminal-id=b
sbx policy log sandbox-alpha
```

**Part 2 — attach the tools... and hit the wall.** Try loading a server into the running sandbox, in **Terminal B**:

```bash terminal-id=b
sbx mcp load wiki-mcp --sandbox sandbox-alpha
```

What you should see — a real error, worth reading:

```output no-run-button no-copy-button
ERROR: add "wiki-mcp" to sandbox "sandbox-alpha": add MCP gateway server: request failed: 409 Conflict: no MCP gateway is running for sandbox "sandbox-alpha"
```

Here's why: each sandbox gets its **MCP gateway at creation time**. `sandbox-alpha` was created back in Step 1 — before MCP was even enabled — so there's no gateway to load into. `sbx mcp load` is for adding *more* servers to a sandbox that already has a gateway.

The fix is the honest one: recreate the sandbox with its tools declared up front. Leave the session in Terminal A (`/exit`), then in **Terminal A**:

```bash terminal-id=a
sbx stop sandbox-alpha
```

```bash terminal-id=a
sbx rm sandbox-alpha
```

Confirm the deletion:

```bash terminal-id=a
y
```

```bash terminal-id=a
sbx create --name sandbox-alpha --static-mcp wiki-mcp --static-mcp duck-mcp claude ~/workshop/sbx
```

Same workspace, same agent — but this one boots with an MCP gateway carrying both servers as its **static set**, "chosen once at creation time" (the `--static-mcp` flag only appears when `SBX_MCP_URL` is set — same gate as 4.1). Your files were never touched: the workspace outlives every sandbox. And now that a gateway exists, `sbx mcp load` can attach *additional* servers to it at runtime — that's what it's for.

Now that a gateway exists, `sbx mcp load <name> --sandbox sandbox-alpha` can also hot-attach *additional* registered servers — it answers `MCP server "..." loaded into sandbox "sandbox-alpha" (live)`, and the agent sees the new tools immediately via MCP's `tools/list_changed` notification. No restart.

**Part 3 — same question, different agent.** Attach and ask — naming the tools you want used:

```bash terminal-id=a
sbx run --name sandbox-alpha
```

```prompt terminal-id=a
Using your MCP tools — wiki-mcp for background and duck-mcp for recent news — tell me what's happening with NASA's Artemis program. At the end, tell me which tools you used, and if anything didn't work, explain what happened.
```

Watch the tool calls: background facts through `wiki-mcp`, recent news through `duck-mcp`. The same kind of lookup that died with a 403 five minutes ago.

> **Why name the tools?** Agents choose their own tools. Claude ships a built-in web search that rides its own API connection — left to itself, it may well use that instead of your MCP servers. Naming tools in the prompt is how you steer; the policy log is how you verify.

**Part 4 — verify at the network layer.** Whatever the agent said it did, check the source of truth in **Terminal B**:

```bash terminal-id=b
sbx policy log sandbox-alpha
```

Three things to read off this log: the Wikipedia 403 from Part 1 is still on the books; the MCP lookups never appear as outbound sandbox traffic (the servers ran on the **host**, through registrations *you* made); and there's a new row — `mcp-gateway.docker.internal:80  forward  <daemon-managed alias>` — the gateway itself, the one audited door between the agent and its tools. The sandbox's network stayed exactly as locked as Step 3 left it.

## 4.4 Wrap up

Leave the session in Terminal A (`/exit`) but **don't tear anything down** — Step 5 reuses `sandbox-alpha`, gateway and all.

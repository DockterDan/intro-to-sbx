# Step 3 — Controlling network access

**Goal:** see the default policy, watch allow vs deny live, read the audit trail, then write your own rules.

:::conditionaldisplay{variable="track" value="guided"}
**Why this matters:** the biggest risk with an agent isn't what it reads — it's what it *sends*. This step puts the room's only phone line through a switchboard you run: calls to numbers on your list connect, everything else is refused, and every attempt — allowed or blocked — lands on an itemized bill you can read afterward. You'll watch a blocked call fail, write your own rules, and check the bill. Even the agent's own calls to its AI provider show up on it.
:::

Same two-tab split as Step 2, and it matters here: **Terminal A** stays attached to the agent the whole step, **Terminal B** is where you run every `sbx policy` command — so you can change the rules *while* the agent is running.

## 3.1 See the defaults

You picked **Balanced** during setup. In **Terminal B**, see what that actually is:

```bash terminal-id=b
sbx policy ls
```

What you should see:

```output no-run-button no-copy-button
POLICY                                 SOURCE   APPLIES TO              SUMMARY
c1abac6c-5f39-4a37-ac68-96f09651b1fc   kit      sandbox:sandbox-alpha   network: 3 allow
local-policy                           local    all                     network: 192 allow; filesystem read: 1 allow; filesystem write: 1 allow
```

Two things to read off this. `local-policy` is your **Balanced** bundle: 192 allow rules covering AI providers, package managers, code hosts, cloud infrastructure, and OS package repos — and everything not on that list is denied by default. Nobody sat down and allowed `reddit.com`; it's simply not covered. And notice the **kit** row: the `claude` agent template itself ships a tiny policy (3 allows — its own API endpoints) scoped to just that sandbox. Even the agent's brain plays by policy rules.

## 3.2 Test the policy from inside

In **Terminal A** (attach with `sbx run --name sandbox-alpha` if you exited), hand the agent five URLs — two on the default lists, three not:

```prompt terminal-id=a
Run these five curls and tell me what you see for each: curl -s https://github.com -o /dev/null -w "github.com: %{http_code} (%{size_download} bytes)\n" ; then the same for pypi.org, reddit.com, wikipedia.org, and nasa.gov.
```

What you should see:

```output no-run-button no-copy-button
github.com: 200 (591919 bytes)
pypi.org: 200 (22848 bytes)
reddit.com: 403 (117 bytes)
wikipedia.org: 403 (120 bytes)
nasa.gov: 403 (115 bytes)
```

Two clean 200s — and three 403s with tiny ~117-byte bodies. That's not reddit answering; it's the **sandbox firewall's block page**. Those connections never left the VM.

## 3.3 The audit trail

Every attempt was recorded, allowed or not. In **Terminal B**:

```bash terminal-id=b
sbx policy log sandbox-alpha
```

Read the **Allowed** section closely: alongside your curls you'll see `api.anthropic.com` with a big count — that's the agent's own API traffic, riding the exact same audited boundary as everything else. This log is the ground truth for the whole workshop: whatever an agent *says* it did, the policy log shows what actually crossed the wire.

We'll need `nasa.gov` later in Step 5, so open just that one domain — a targeted allow, not "turn the firewall off". In **Terminal B**:

```bash terminal-id=b
sbx policy allow network nasa.gov
```

What you should see:

```output no-run-button no-copy-button
Rule added to policy local (scope: global): 3fa2c8d1-77e0-42a3-a162-536ec0bceeed (nasa.gov)
```

## 3.4 Add a custom deny rule

Deny beats allow. Block a domain that IS on the default allow list — `pypi.org`. In **Terminal B**:

```bash terminal-id=b
sbx policy deny network pypi.org
```

```bash terminal-id=b
sbx policy ls
```

The rule lands inside `local-policy` — watch its summary change:

```output no-run-button no-copy-button
local-policy   local   all   network: 193 allow, 1 deny; filesystem read: 1 allow; filesystem write: 1 allow
```

193 allows now (your nasa.gov rule joined the bundle), and **1 deny** — the one you just wrote.

## 3.5 Verify it's enforced

Rules apply immediately on most setups. If one ever doesn't take (say, mid-connection), bounce the sandbox: `sbx stop sandbox-alpha`, then `sbx run --name sandbox-alpha`.

In **Terminal A**, ask the agent to hit PyPI both ways:

```prompt terminal-id=a
Run curl -s https://pypi.org -o /dev/null -w "pypi.org: %{http_code} (%{size_download} bytes)\n" and then run pip install requests 2>&1 | tail -3. Tell me what you see for each.
```

What you should see:

```output no-run-button no-copy-button
pypi.org: 403 (171 bytes)

ERROR: Could not find a version that satisfies the requirement requests
ERROR: No matching distribution found for requests
```

`pip` didn't break — its network did. The agent's toolchain works exactly as far as your policy lets it, and not a request further.

## 3.6 Remove a rule

Practice on a throwaway so we don't disturb the rules we need. In **Terminal B** — add, confirm, remove, confirm:

```bash terminal-id=b
sbx policy deny network example.com
```

```bash terminal-id=b
sbx policy ls
```

```bash terminal-id=b
sbx policy rm network --resource example.com
```

```bash terminal-id=b
sbx policy ls
```

Removal prints `Rule removed from policy local: resources=example.com`, and the summary drops back to `1 deny`.

> **Subdomains and wildcards:** `nasa.gov` and `www.nasa.gov` are *different hosts* to the policy engine — an allow for one doesn't cover the other. The rule syntax handles this: wildcard subdomains (`*.nasa.gov`), port suffixes (`nasa.gov:443`), comma lists, `**` for allow-everything, and `--sandbox <name>` to scope a rule to one sandbox instead of all. See `sbx policy allow network --help`.

> **Leave things as they are now** going into Step 4: `nasa.gov` allowed, `pypi.org` denied. Both are load-bearing later.

Allow list, deny list, audit log — you've now touched the whole enforcement surface. Next, we give the agent actual tools.

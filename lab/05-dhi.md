# Step 5 — Hardening the agent's output with DHI

**Goal:** give the agent an open-ended build task, watch the policy boundary shape its research, then build the result, scan it, swap the base image for a **Docker Hardened Image**, and compare the CVE counts.

:::conditionaldisplay{variable="track" value="guided"}
**Why this matters:** now the agent does real work — researching and writing a small web app — and you'll see the switchboard shape what it produces: sources it couldn't call, it worked around, and it documented that honestly. Then comes the part most people skip: before anything ships, you inspect the package it ships in. The scanner counts known security flaws in the packaging, and swapping one line for Docker's pre-hardened packaging makes the worst of them vanish. The agent writes the product; you control the shipping crate.
:::

Same split: agent work in **Terminal A**, everything `docker` and `sbx policy` in **Terminal B**.

## Before you start, re-open PyPI

The agent may want packages while researching, and our Step 3 deny rule would break that for reasons unrelated to the demo. In **Terminal B** — same mechanic as 3.6:

```bash terminal-id=b
sbx policy rm network --resource pypi.org
```

```bash terminal-id=b
sbx policy ls
```

`nasa.gov` stays allowed — that one is about to matter.

## 5.1 Project setup

In **Terminal B**, create the project directory and the starter Dockerfile:

```bash terminal-id=b
mkdir -p ~/workshop/sbx/research-app
```

Press **Save** to write the Dockerfile into `research-app/`:

```dockerfile save-as=sbx/research-app/Dockerfile terminal-id=b
FROM python:3.14-slim
WORKDIR /app
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
```

Note what's *not* there yet: no `app.py`. The agent writes that.

## 5.2 The build task

In **Terminal A**, attach to the agent:

```bash terminal-id=a
sbx run --name sandbox-alpha
```

Give it the job — open-ended on purpose:

```prompt terminal-id=a
There's a research-app/ directory in the workspace with a Dockerfile already in it. Build me a small, polished dashboard that mixes two threads: open-source projects from GitHub and current space missions, connected in an unexpected way. Research with real sources — try nasa.gov, space news outlets, and GitHub — cap yourself at 5-6 external requests and move on when something is blocked. Constraints: only write app.py; stdlib only; embed all research data inline; serve HTTP on port 8000 on 0.0.0.0; light clean design, no emoji; include a Sources section. Sanity-check it with curl before reporting back.
```

## 5.3 Watch the boundary do its job

While the agent researches, check what's actually crossing the network in **Terminal B**:

```bash terminal-id=b
sbx policy log sandbox-alpha
```

`nasa.gov` and `github.com`: allowed. `www.jpl.nasa.gov`: blocked — subdomains are their own hosts, remember — so the agent lost a source mid-research, adapted, and kept working. **The policy shaped the work product.** (Also in the blocked list: `datadoghq.com` — that's Claude Code's own telemetry being denied. *Everything* obeys the policy, including the agent's phone-home.)

When the agent reports back, look at what it built (Terminal B):

```bash terminal-id=b
cat sbx/research-app/app.py
```

## 5.4 Build and run it

The agent wrote the code; the container is yours to build. In **Terminal B**:

```bash terminal-id=b
cd ~/workshop/sbx/research-app
```

```bash terminal-id=b
docker build -t research-app:v1 .
```

```bash terminal-id=b
docker run -d --rm -p 8000:8000 --name research-app research-app:v1
```

Hit it:

```bash terminal-id=b
curl -s http://localhost:8000 | head -8
```

There's your dashboard — note the "Sources" section, where the agent documented what the sandbox let it reach. Stop the container:

```bash terminal-id=b
docker stop research-app
```

## 5.5 Scan it with Docker Scout

What did we just ship? In **Terminal B**:

```bash terminal-id=b
docker scout quickview research-app:v1
```

What you should see (counts vary in real life):

```output no-run-button no-copy-button
 Target             │  research-app:v1  │    1C     2H     3M    28L     7?
   digest           │  3ab201f359e1     │
 Base image         │  python:3-slim    │    1C     2H     3M    28L     7?
 Updated base image │  python:alpine    │    0C     0H     0M     0L
```

Read the two count columns: the target's vulnerabilities and the base image's vulnerabilities are **identical**. Every CVE came from the base — not from the code the agent wrote. See the detail:

```bash terminal-id=b
docker scout cves research-app:v1
```

## 5.6 Swap the base for a Docker Hardened Image

DHI images live under the `dhi.io` namespace — one-time login, then change exactly **one line** of the Dockerfile. In **Terminal B**:

```bash terminal-id=b
docker login dhi.io
```

```bash terminal-id=b
sed -i.bak 's|FROM python:3.14-slim|FROM dhi.io/python:3.14|' Dockerfile
```

Rebuild as `:v2`, run it once to confirm nothing changed, then rescan:

```bash terminal-id=b
docker build -t research-app:v2 .
```

```bash terminal-id=b
docker run -d --rm -p 8000:8000 --name research-app research-app:v2
```

```bash terminal-id=b
docker stop research-app
```

```bash terminal-id=b
docker scout quickview research-app:v2
```

What you should see:

```output no-run-button no-copy-button
 Target   │  research-app:v2  │    0C     0H     3M    15L    18?
   digest │  cc23ed01d069     │
```

**Zero criticals, zero highs.** The remaining mediums/lows are unfixed upstream issues that every Python image carries — nothing with a fix is left unpatched.

## 5.7 Compare the two images

```bash terminal-id=b
docker scout compare --to research-app:v1 research-app:v2
```

Three things in that diff are the whole DHI pitch. The vulnerability row: criticals and highs go to **zero**. The size rows: 42 fewer packages, 13 MB smaller — attack surface that simply left the building. And the **Labels** section: `com.docker.dhi.compliance=cis`, an explicit end-of-life date, versioned provenance — the compliance paperwork ships *inside* the image.

Same code, same Dockerfile shape, same runtime behavior — one `FROM` line changed. **The agent ships the app; DHI hardens what it ships.**

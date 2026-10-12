# Jenkins Hands-On: A CI Pipeline from Scratch

A hands-on, phase-by-phase guide to learning **Jenkins** by building a real CI pipeline for a web application.
Everything runs locally with Docker — no cloud account required.

**Sample application:** the guide uses a small **Vite + React** app (a portfolio site) that exposes `npm run lint`
and `npm run build` (producing `dist/`). It is referred to throughout as **`my-portfolio`**. Substitute your own
repository, branch, and build commands — the Jenkins concepts are identical.

> **Placeholders:** in the steps below, `my-portfolio` is the example app, and `<your-username>` / `<your-repo>` stand
> in for your own GitHub account and repository. Replace them with your real values as you go.

## What we're building

A local, self-hosted Jenkins **CI system** (controller + dedicated agent) that automatically lints and builds the
sample app on every push, and publishes the production `dist/` bundle as a downloadable build artifact.

**Focus:** Core CI only — install, pipeline, build/lint, artifacts, credentials, GitHub integration.
Deployment (CD) is intentionally out of scope.

**Cost:** $0 running locally via Docker. Cheap cloud options are listed at the bottom if you later want an always-on box.

---

## Concepts you'll learn

| Term | What it means |
|------|---------------|
| **Controller** | The Jenkins server that schedules jobs, serves the UI, stores config & history. |
| **Agent (node)** | A machine/container that actually executes builds. Keeps the controller clean. |
| **Inbound agent** | Agent that dials *into* the controller (uses a secret) — ideal for Docker. |
| **Job / Item** | A unit of work. We use **Pipeline** jobs (not legacy Freestyle). |
| **Pipeline** | A scripted sequence of stages defined in a `Jenkinsfile` stored in your repo. |
| **Declarative Pipeline** | The structured `pipeline { stages { ... } }` syntax — friendlier than Groovy Scripted. |
| **Jenkinsfile** | The file in your repo that *is* the pipeline (Pipeline as Code). |
| **Stage** | A labeled logical step (Checkout, Install, Lint, Build). |
| **Step** | A single action inside a stage (`sh`, `checkout`, `archiveArtifacts`). |
| **Agent directive** | `agent { ... }` — which node/label/container a stage runs on. |
| **Tool** | A managed binary (e.g. Node.js LTS) auto-installed by Jenkins on an agent. |
| **Credentials** | Encrypted secrets (PAT, SSH key) referenced by ID — never hardcoded. |
| **Artifact** | A file produced by a build and archived by Jenkins (`dist/`). |
| **Jenkins Home** | `$JENKINS_HOME` — the directory holding jobs, plugins, users. Must persist. |
| **Plugin** | Extension (Git, NodeJS, Pipeline, Docker…) installed from the Update Center. |
| **Webhook / Poll SCM** | How GitHub tells Jenkins "there's a new commit, build it". |
| **JCasC** | Configuration-as-Code plugin — optional, configures Jenkins from a YAML file. |

---

## Architecture

```
                    ┌───────────────────────────────────────────────┐
   GitHub           │            Docker network: jenkins-net         │
   your-username/   │                                                │
   my-portfolio ─(webhook/poll)─► ┌────────────────────────────┐     │
       ▲            │             │  jenkins-controller         │    │
       │            │             │  jenkins/jenkins:lts-jdk21  │    │
       │            │             │  ports: 8080, 50000         │    │
       │            │             │  volume: jenkins_home       │    │
       │            │             └───────────┬────────────────┘    │
       │            │                         │ schedules build      │
       │            │                         ▼                      │
       │            │             ┌────────────────────────────┐     │
       │            │             │  jenkins-agent              │    │
       │            │             │  jenkins/inbound-agent      │    │
       │            │             │  runs: checkout, npm ci,    │    │
       │            │             │        lint, build          │    │
       │            │             │  tools: Node.js LTS, Git    │    │
       │            │             └────────────────────────────┘     │
       │            │  artifact: dist/  ────────────────┐           │
       │            └────────────────────────────────────┼───────────┘
       │                                                 │
   you open http://localhost:8080  ◄── logs, download artifact
```

**Why controller + agent?**
- Mirrors how real Jenkins is deployed (controller stays responsive; builds are isolated).
- The agent's filesystem is disposable — you can nuke the agent without losing Jenkins config.
- Teaches the controller/agent mental model early, which is the #1 source of confusion.

**Why Declarative Pipeline + a `Jenkinsfile`?**
- Pipeline config lives in Git — reviewable, versioned, reproducible.
- Freestyle jobs can't be code-reviewed and don't scale.

---

## Project structure

```
jenkins-learning/                 ← the learning workspace (this repo)
├── docs/
│   ├── plan.md                   ← this file
│   ├── phase-1-foundation.md
│   ├── phase-2-first-pipeline.md
│   ├── phase-3-quality-gates.md
│   └── phase-4-github-credentials.md
├── docker-compose.yml            ← controller + agent (Phase 1)
├── .env                          ← agent secret etc., git-ignored (Phase 1)
├── .env.example                  ← committed template (Phase 1)
├── jenkins/
│   └── Dockerfile                ← optional custom agent image (Phase 3)
└── .gitignore

my-portfolio/                     ← the app you're wiring into CI (separate repo)
└── Jenkinsfile                   ← added in Phase 2
```

---

## Data flow

1. You `git push` to `my-portfolio`.
2. GitHub webhook (or periodic SCM poll) notifies the controller.
3. Controller queues the Pipeline, dispatches to the agent.
4. Agent checks out the repo, runs `npm ci` → `npm run lint` → `npm run build`.
5. Agent archives `dist/` as a Jenkins artifact.
6. Controller shows the result (green/red), logs, and the downloadable artifact.
7. (Optional Phase 4) Jenkins reports the commit status back to GitHub on the PR.

---

## Phase roadmap (Core CI — 4 phases)

### Phase 1 — Foundation: Jenkins up in Docker
Stand up `jenkins-controller` + `jenkins-agent` via Docker Compose with persistent volumes, complete the
unlock wizard, install the plugin set we need, create your admin user, and confirm the agent shows as **online**.
**Deliverable:** Jenkins UI reachable at `http://localhost:8080`, one connected agent.

### Phase 2 — First Pipeline: build the sample app
Add the first `Jenkinsfile` to `my-portfolio`, configure the managed **Node.js LTS** tool, and create a Declarative
Pipeline job that checks out the repo, installs deps (`npm ci`), and builds (`npm run build`). Archive `dist/`.
**Deliverable:** a green build whose artifact you can download from the Jenkins UI.

### Phase 3 — Quality gates & structure
Refactor into a real pipeline: parallel **Lint** + **Build** stages, `post { always/success/failure }` blocks,
timestamps, build description, fail-fast semantics, and a custom agent image that pre-bakes Node + Git.
**Deliverable:** lint reports as an **advisory** red stage (non-blocking, useful when a project isn't lint-clean yet);
a successful build archives the artifact and annotates the build.

### Phase 4 — GitHub integration & credentials
Create a GitHub Personal Access Token / SSH key as a Jenkins **credential** (by ID), switch the job to use it,
wire up a webhook (via `ngrok` for localhost) or enable SCM polling, and add a CI status badge to the README.
**Deliverable:** pushing to `my-portfolio` automatically triggers a build and shows a status on GitHub.

### Stretch (optional, later)
- Add a **Deploy** stage (e.g. Vercel/Netlify CLI + a stored token) to turn this into full CI/CD.
- Jenkins Configuration as Code (JCasC) so the whole controller config is versioned.
- Build a release tarball and publish to GitHub Releases.

---

## Cheap cloud options (only if/when you want an always-on Jenkins)

| Provider | Offering | Approx. cost | Notes |
|----------|----------|--------------|-------|
| **Oracle Cloud Free Tier** | 4 ARM cores / 24 GB RAM VM | **$0 forever** | Best free option; ARM is fine for Jenkins. |
| **Hetzner Cloud** | CX22 (2 vCPU / 4 GB) | ~€4–5 / mo | Cheapest solid paid VM; EU regions. |
| **DigitalOcean** | Basic droplet 2 GB | ~$6 / mo | Simple, good docs, US/EU. |
| **Fly.io / Railway** | Small always-on machine | ~$5 / mo | Easy deploys; less disk for Jenkins home. |
| **AWS EC2** | `t3.small`/`t4g.small` | ~$8–15 / mo | More setup; watch EBS + data transfer. |

> For all cloud options add ~$1–2/mo for a persistent volume (Jenkins home) and a domain/HTTPS if exposing publicly.
> **Security note:** never expose Jenkins directly on the internet without HTTPS + auth; prefer SSH tunnels or a VPN.

---

## Prerequisites

- Docker Desktop (or Docker Engine + Compose) installed and running.
- ~4 GB RAM free for the Jenkins containers.
- A Git repository you can push to (the sample app).
- `ngrok` (free account) — only needed in Phase 4 for webhooks.

---

## How to use this guide

- This is a hands-on guide: type the code yourself rather than copy-pasting blindly — that's where the learning is.
- Work **one phase at a time**. Don't advance until the current phase builds and its **Test** section passes.
- Each phase starts from the previous phase's working state.
- Keep secrets out of Git: `.env` is git-ignored; credentials live only inside Jenkins.

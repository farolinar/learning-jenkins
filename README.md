# Jenkins Hands-On: A CI Pipeline from Scratch

Learn **Jenkins** the practical way — by building a real CI pipeline for a web app, entirely on your own machine
with Docker. **No cloud account, no cost.**

This repo is a guided, phase-by-phase tutorial. You stand up a Jenkins **controller + agent**, wire it to a
GitHub repository, and grow a `Jenkinsfile` from a hello-world build into a genuine CI pipeline with parallel quality
gates, artifacts, and encrypted credentials.

> **Sample app:** the guide uses a small **Vite + React** app (a portfolio site) referred to as `my-portfolio`, with
> `npm run lint` and `npm run build` (output in `dist/`). Any Node/JS project works — swap in your own repo, branch,
> and build commands.

---

## What you'll learn

- Jenkins **controller / agent** architecture and why builds run on agents
- **Pipeline as Code** with a Declarative `Jenkinsfile`
- Managed **build tools** (Node.js LTS) provisioned by Jenkins
- **Parallel stages**, `post` conditions, fail-fast, timeouts, and build retention
- **Artifact** archiving and fingerprinting
- **Credentials** (PAT/SSH) stored securely and referenced by ID
- **GitHub integration** via webhooks (or SCM polling) and commit status
- Running the whole thing **locally with Docker Compose** — and cheap options if you later want an always-on box

---

## Prerequisites

- **Docker Desktop** (or Docker Engine + Compose) installed and running
- ~4 GB of free RAM for the Jenkins containers
- A **Git** repository you can push to (to follow Phase 4)
- `ngrok` (free account) — only needed in Phase 4 for local webhooks

---

## Curriculum

Work through the phases **in order**; each one starts from the previous phase's working state.

| # | Phase | What you build | Guide |
|---|-------|----------------|-------|
| 0 | Plan | Architecture, concepts, and roadmap | [`docs/plan.md`](docs/plan.md) |
| 1 | Foundation | Jenkins controller + inbound agent in Docker | [`docs/phase-1-foundation.md`](docs/phase-1-foundation.md) |
| 2 | First pipeline | A `Jenkinsfile` that checks out, installs, and builds the app | [`docs/phase-2-first-pipeline.md`](docs/phase-2-first-pipeline.md) |
| 3 | Quality gates | Parallel lint/build, `post` conditions, custom agent image | [`docs/phase-3-quality-gates.md`](docs/phase-3-quality-gates.md) |
| 4 | GitHub integration | Credentials, webhook/SCM polling, commit status, CI badge | [`docs/phase-4-github-credentials.md`](docs/phase-4-github-credentials.md) |

Start here: **[`docs/phase-1-foundation.md`](docs/phase-1-foundation.md)** — or read **[`docs/plan.md`](docs/plan.md)**
first for the big picture.

---

## Quick start

```bash
git clone https://github.com/<your-username>/jenkins-learning.git
cd jenkins-learning

# 1. Create your local env file (git-ignored)
cp .env.example .env

# 2. Start the Jenkins controller
docker compose up -d controller

# 3. Get the initial admin password, then open http://localhost:8080
docker exec jenkins-controller cat /var/jenkins_home/secrets/initialAdminPassword
```

In the Jenkins UI: **Install suggested plugins**, create your admin user, then follow
[Phase 1](docs/phase-1-foundation.md) to create the `docker-agent` node, paste its secret into `.env`, and start the
agent:

```bash
docker compose up -d --build agent
docker compose ps          # both containers should be Up
```

---

## Architecture

```
   GitHub  ──(webhook / poll)──►  jenkins-controller  ──►  jenkins-agent
 my-portfolio                     (UI, jobs, config)      (runs checkout,
                                      :8080                 npm ci, lint, build)
                                      volume: jenkins_home   tools: Node.js LTS, Git
                                                             artifact: dist/
```

- The **controller** schedules work, serves the UI, and stores all state in the `jenkins_home` volume.
- The **agent** is a disposable worker that actually executes the pipeline.

See [`docs/plan.md`](docs/plan.md) for the full diagram and concept glossary.

---

## Project structure

```
jenkins-learning/
├── docs/                       # the tutorial (start with plan.md)
├── docker-compose.yml          # controller + agent (Phase 1)
├── .env.example                # template for the agent secret
├── jenkins/
│   └── Dockerfile              # custom agent image (Phase 3)
└── README.md
```

Your application repository (e.g. `my-portfolio/`) stays separate; the `Jenkinsfile` lives in its root.

---

## Cost

Running locally is **$0**. If you later want an always-on Jenkins, the cheapest options are Oracle Cloud
Free Tier ($0, 4 ARM cores / 24 GB), Hetzner CX22 (~€4–5/mo), and DigitalOcean (~$6/mo). See the table in
[`docs/plan.md`](docs/plan.md#cheap-cloud-options-only-ifwhen-you-want-an-always-on-jenkins).

---

## Notes

- Keep secrets out of Git. `.env` and `*.code-workspace` are git-ignored; credentials live only inside Jenkins.
- Placeholders like `<your-username>` and the name `my-portfolio` stand in for your own values.
- If your project is already lint-clean, remove the advisory `catchError` wrapper in the Phase 3 pipeline to make
  lint a hard gate.

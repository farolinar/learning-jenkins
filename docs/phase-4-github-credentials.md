# Phase 4 — GitHub integration & credentials

## Goal
Connect the `my-portfolio` repository to Jenkins so that **pushing automatically triggers a build**, authenticating with a
stored **credential** (never a hardcoded secret), and optionally reporting the build result back to GitHub as a commit
status + README badge.

**End state:** `git push` to `my-portfolio` starts a Jenkins build within seconds, with no manual "Build Now".

---

## Concepts

| Concept | Why it matters here |
|---------|---------------------|
| **Credential (by ID)** | Secrets are stored encrypted in Jenkins and referenced by ID — never in the Jenkinsfile. |
| **Username/password credential** | What Git-over-HTTPS needs (username + PAT as password). |
| **Secret text credential** | What the GitHub API integration needs (a token). |
| **Webhook** | GitHub pushes an event to Jenkins so builds start instantly. |
| **`pollSCM`** | Fallback: Jenkins periodically checks Git. Survives without a public URL. |
| **`GitHubCommitStatusSetter`** | Pipeline step that posts pass/fail back to the commit. |
| **Personal Access Token (PAT)** | A scoped GitHub token that acts as your password for automation. |
| **ngrok** | Gives your localhost Jenkins a temporary public HTTPS URL for webhooks. |

---

## Dependencies
- Phase 3 complete: quality-gated pipeline runs green.
- GitHub account with admin rights on `your-username/my-portfolio`.
- `ngrok` installed ([ngrok.com/download](https://ngrok.com/download)) and authenticated (free account).

---

## Security warning (read first)
Tunneling Jenkins with `ngrok` exposes it to the **public internet**. Your admin login protects the UI, but
**stop the tunnel when you're done** (`Ctrl+C`). Never point a public tunnel at a Jenkins with anonymous read access or
weak credentials. For production, use HTTPS + a reverse proxy + firewall allow-lists.

---

## Step 1 — Create a GitHub Personal Access Token

Use a **classic** token (simplest for this lesson):

1. GitHub → **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)**.
2. Scopes:
   - `repo` — read (and for private repos, write) repository contents & commit statuses.
   - `admin:repo_hook` — lets Jenkins manage/verify webhooks (optional).
3. Generate and **copy** the token (`ghp_...`). You won't see it again.

---

## Step 2 — Store credentials in Jenkins

**Manage Jenkins → Credentials → System → Global credentials (unrestricted) → + Add Credentials.**

Create **two** credentials:

**A. For Git checkout (username + password)**
- Kind: **Username with password**
- Username: your GitHub username
- Password: the PAT
- ID: `github-pat`
- Description: `GitHub PAT for Git checkout`

**B. For the GitHub API (secret text)**
- Kind: **Secret text**
- Secret: the same PAT
- ID: `github-token`
- Description: `GitHub API token`

**Explain:** the Jenkinsfile and the job reference these by **ID** only. The secret itself lives encrypted in
`$JENKINS_HOME/credentials.xml` (masked in logs).

---

## Step 3 — Point the job at the credential

Edit the `my-portfolio-ci` job → **Pipeline → SCM**:
- **Credentials:** select `github-pat — GitHub PAT for Git checkout`
- Keep **Repository URL:** `https://github.com/your-username/my-portfolio.git`
- **Save**

The next build will authenticate using the stored credential (and avoid anonymous rate limits).

---

## Step 4 — Trigger builds on push (choose one)

### Option A — Webhook (instant; needs a public URL)

Start a tunnel to Jenkins:

```bash
ngrok http 8080
```

Copy the **Forwarding** HTTPS URL, e.g. `https://abcd-1234.ngrok-free.app`.

In GitHub: repo → **Settings → Webhooks → Add webhook**:
- **Payload URL:** `https://<your-ngrok-url>/github-webhook/`  ← **trailing slash matters**
- **Content type:** `application/json`
- **Which events:** *Just the push event* (add *Pull requests* if you like)
- **Active:** ✅ → **Add webhook**

In Jenkins: edit `my-portfolio-ci` → **Build Triggers** → ✅ **GitHub hook trigger for GITScm polling** → **Save**.

Test: make a trivial commit and `git push`. A build should start within seconds. Check the webhook's
"Recent Deliveries" tab in GitHub for a `200` response.

### Option B — SCM polling (no public URL needed)

Add a `triggers` block to `my-portfolio/Jenkinsfile`:

```groovy
pipeline {
    agent { label 'docker' }
    tools { nodejs 'node24' }

    triggers {
        pollSCM('H/5 * * * *')     // check GitHub ~every 5 minutes (H = load-balanced)
    }
    // ...rest unchanged
}
```

Commit + push the Jenkinsfile change. Jenkins will now poll and start a build when it detects new commits.
Slower than a webhook, but works behind NAT with no tunnel.

> You can use **both**; the webhook gives instant feedback and polling is a safety net.

---

## Step 5 — (Optional) Report status back to the commit

**Why:** a green ✓ / red ✗ appears on the commit in GitHub, which is what teammates actually look at.

**5a. Configure a GitHub Server** (one-time):
**Manage Jenkins → System → GitHub → Add GitHub Server → GitHub Server**:
- **Name:** `github`
- **API URL:** `https://api.github.com`
- **Credentials:** add → **Secret text** → select `github-token`
- ✅ **Manage hooks** (optional) → **Test connection** should succeed → **Save**.

**5b. Publish status from the pipeline.** Extend the `post` section of `my-portfolio/Jenkinsfile`:

```groovy
    post {
        success {
            archiveArtifacts artifacts: 'dist/**', fingerprint: true
            setGitHubStatus('SUCCESS', 'CI passed')
        }
        failure {
            setGitHubStatus('FAILURE', 'CI failed')
        }
        // ...keep the rest (always, cleanup) from Phase 3
    }
```

And add this helper **above** `pipeline { }` (or at the bottom of the file):

```groovy
void setGitHubStatus(String state, String description) {
    step([
        $class: 'GitHubCommitStatusSetter',
        contextSource: [$class: 'ManuallyEnteredCommitContextSource',
                         context: 'continuous-integration/jenkins'],
        reposSource:   [$class: 'ManuallyEnteredRepositorySource',
                        url: 'https://github.com/your-username/my-portfolio'],
        statusResultSource: [$class: 'ConditionalStatusResultSource',
            results: [[$class: 'AnyBuildResult', state: state, message: description]]]
    ])
}
```

> This uses `GitHubCommitStatusSetter` (from the **GitHub** plugin, installed in Phase 1). It needs the GitHub Server
> from 5a. If the status doesn't appear, check **Manage Jenkins → System Log** for API/auth errors.

---

## Step 6 — Add a CI badge to the README

If you keep `ngrok` running (public URL), add to `my-portfolio/README.md`:

```markdown
![Jenkins CI](https://img.shields.io/jenkins/build?jobUrl=https%3A%2F%2F<your-ngrok-url>%2Fjob%2Fmy-portfolio-ci)
```

For a stable badge you'd need a permanently reachable Jenkins (see cloud options in `plan.md`). While local-only,
the badge won't render on GitHub.com — it's a preview of what a hosted CI badge looks like.

---

## Test — verify Phase 4 works

1. `git push` a trivial change to `my-portfolio` (e.g. edit README).
2. Jenkins starts a build within seconds (webhook) or ≤5 min (polling). ✅
3. Console output shows the checkout used the credential (no anonymous warning). ✅
4. (If Step 5 done) GitHub shows a commit status; a passing build turns it green. ✅
5. Stop `ngrok` when finished and confirm builds fall back to `pollSCM` (if Option B enabled). ✅

| Symptom | Cause | Fix |
|---------|-------|-----|
| Webhook delivery shows `404` | Missing trailing `/` or wrong path | URL must end `/github-webhook/` |
| Webhook `200` but no build | Trigger checkbox off | Enable "GitHub hook trigger for GITScm polling" |
| `403` on checkout | PAT expired / wrong scope | Regenerate PAT with `repo` scope, update `github-pat` |
| Commit status never appears | GitHub Server not configured / wrong credential | Redo Step 5a; check System Log |
| Badge broken | Jenkins not publicly reachable | Expected locally; needs hosted Jenkins |

---

## You did it — Core CI complete

You now have a working CI system covering: controller/agent architecture, Pipeline as Code, managed tools, parallel
quality gates, artifact archiving, encrypted credentials, and GitHub-triggered builds.

**Where to go next (stretch):**
- **Deploy stage:** add a `Deploy` stage using the Vercel CLI + a `VERCEL_TOKEN` secret (turns this into CI/CD).
- **Multibranch Pipeline:** auto-discover branches/PRs and build each.
- **Configuration as Code (JCasC):** version the entire controller config in `jenkins.yaml`.
- **Move to a cloud VM:** see the cheap options table in `docs/plan.md` for an always-on Jenkins.

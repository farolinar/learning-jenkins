# Phase 2 — First Pipeline: build the sample app

## Goal
Add a **Declarative `Jenkinsfile`** to `my-portfolio`, configure the managed **Node 24 LTS** tool, and create a Pipeline
job that checks out the repo, installs dependencies, and runs the production build — then archives `dist/` as a
downloadable artifact.

**End state:** a green build for `my-portfolio-ci` with `dist/**` available under "Build Artifacts".

---

## Concepts

| Concept | Why it matters here |
|---------|---------------------|
| **Jenkinsfile** | The pipeline stored in your repo — config as code, reviewable in PRs. |
| **Declarative Pipeline** | The `pipeline { }` syntax: predictable, structured, recommended for teams. |
| **`agent { label }`** | Pins the pipeline to a specific node label (`docker linux`). |
| **`tools { nodejs }`** | Tells Jenkins to provision a managed Node.js version and put it on `PATH`. |
| **`checkout scm`** | Checks out the same repo/branch Jenkinsfile that defines this pipeline. |
| **`sh` step** | Runs a shell command. `sh 'node --version'` prints the tool version. |
| **`archiveArtifacts`** | Saves build outputs so you can download them later. |
| **Pipeline from SCM** | Job definition that loads the `Jenkinsfile` from Git rather than the UI. |

---

## Dependencies
- Phase 1 complete: controller + `docker-agent` online, **NodeJS** plugin installed.

---

## Step 1 — Add the `Jenkinsfile` to `my-portfolio`

Create a file named exactly `Jenkinsfile` (no extension) in the **root** of the `my-portfolio` repo:

```groovy
// Jenkinsfile
pipeline {
    agent { label 'docker' }        // run on our agent (label set in Phase 1)

    tools {
        nodejs 'node24'             // provision the managed tool named "node24" (see Step 2)
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm        // fetch the repo that contains this Jenkinsfile
            }
        }

        stage('Install') {
            steps {
                sh 'node --version' // proves the managed tool is on PATH
                sh 'npm --version'
                sh 'npm ci'         // clean, lockfile-exact install (preferred in CI over npm install)
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'  // runs "vite build" -> produces dist/
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: 'dist/**', fingerprint: true
        }
    }
}
```

**Explain:**
- `npm ci` (not `npm install`) is the CI-correct command: it deletes `node_modules` and installs exactly what
  `package-lock.json` pins, so builds are reproducible.
- `tools { nodejs 'node24' }` references a tool *named* `node24` that you configure in Step 2 — the name is arbitrary
  but must match.
- We only archive on `success`; a broken build produces no artifacts.

Commit and push it:

```bash
cd my-portfolio
git add Jenkinsfile
git commit -m "ci: add Jenkins pipeline"
git push origin main
```

---

## Step 2 — Create the managed Node.js tool

**Why:** the NodeJS plugin downloads a Node runtime and injects it into the build `PATH` — no Node needed on the host.

1. **Manage Jenkins → Tools → NodeJS installations → Add NodeJS**.
2. **Name:** `node24` (must match the `Jenkinsfile`).
3. **Install automatically:** checked.
4. **Version:** `24.21.0` (or the latest **24.x LTS** offered).
5. **Save**.

> **Why pin an LTS and not the newest release?** The dropdown lists *every* nodejs.org release, including the
> **Current** line. Current is the staging ground for the next LTS — breaking changes, short support, no security
> backports. CI should pin an **LTS** line (long support + backports). Node 24 was the **Active LTS** at the time of
> writing; check the [Node.js release schedule](https://github.com/nodejs/release#release-schedule) and use whichever
> line is the current Active LTS for you.

> Gotcha: if the version dropdown is empty, the update center is unreachable from the controller. Verify the container
> has internet: `docker exec jenkins-controller curl -sI https://updates.jenkins.io | head -1`.

---

## Step 3 — Create the Pipeline job

1. **New Item** → name `my-portfolio-ci` → select **Pipeline** → **OK**.
2. Scroll to **Pipeline**:
   - **Definition:** `Pipeline script from SCM`
   - **SCM:** `Git`
   - **Repository URL:** `https://github.com/your-username/my-portfolio.git`
   - **Branch Specifier:** `*/main`
   - **Script Path:** `Jenkinsfile`
3. Leave credentials empty — the repo is public.
4. **Save**.

**Explain:** The job itself contains almost nothing; the real pipeline is the `Jenkinsfile` in Git. Changing CI behavior
means changing code and opening a PR — that's the payoff of Pipeline as Code.

---

## Test — verify Phase 2 works

1. On the job page click **Build Now**.
2. Open the build → **Console Output**. You should see:
   - `Checkout`, `Install`, `Build` stages each pass.
   - A printed Node version starting with `v24`.
   - `vite build` finishing with `✓ built in ...`.
   - `Finished: SUCCESS`.
3. On the build page, a **Build Artifacts** section appears — download/verify `dist/` is listed. ✅

Common failures and fixes:
| Symptom | Cause | Fix |
|---------|-------|-----|
| `node: command not found` | Tool name mismatch or NodeJS plugin missing | Name must be `node24`; install NodeJS plugin |
| `npm ci ... EBADENGINE` | Node 24 vs a dependency wanting another major | Check `engines` in package.json; try Node 22 LTS |
| Job stuck "pending" | Agent offline / wrong label | Node online? Label includes `docker`? |
| `not a git repository` / checkout fails | Wrong branch spec | Use `*/main` |

**Next:** Phase 3 hardens the pipeline with quality gates, parallel stages, and structured post-conditions.

# Phase 3 — Quality gates & structure

## Goal
Turn the toy pipeline into a real one: run **Lint** and **Build** in parallel, fail fast, add structured
`post`-conditions, timestamps, build retention, a build description, and a custom agent image with the CLI tools
we want.

**End state:** a polished pipeline where lint shows as an **advisory** red stage (non-blocking), a genuine build failure
turns the build **red** with no artifact archived, and console output is timestamped.

---

## Concepts

| Concept | Why it matters here |
|---------|---------------------|
| **`options { }`** | Pipeline-wide behavior: timestamps, retention, concurrency, fail-fast. |
| **`environment { }`** | Inject variables (e.g. `CI=true`) for all steps. |
| **`parallel { }`** | Run independent stages simultaneously to cut wall-clock time. |
| **`failFast true`** | Abort the other parallel branches as soon as one fails. |
| **`catchError`** | Catch a step failure and decide the stage/build result — used to keep lint advisory. |
| **`post { always/success/failure/cleanup }`** | Guaranteed cleanup + status-specific reporting. |
| **`buildDiscarder`** | Keep only N recent builds (protects the Jenkins home volume from filling up). |
| **`fingerprint: true`** | Track which build produced which artifact. |
| **Custom agent image** | Bake extra CLI tools into the worker so pipelines don't install them every run. |

---

## Dependencies
- Phase 2 complete: `my-portfolio-ci` builds green with Node 24.

---

## Step 1 — Upgrade the `Jenkinsfile`

Replace the contents of `my-portfolio/Jenkinsfile` with:

```groovy
// Jenkinsfile
pipeline {
    agent { label 'docker' }                 // top-level agent shared by all stages

    tools { nodejs 'node24' }                // managed Node 24 (configured in Phase 2)

    options {
        timestamps()                         // prepend timestamps to console output (Timestamper plugin)
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        disableConcurrentBuilds()            // one build at a time for this job
        parallelsAlwaysFailFast()            // if any parallel branch fails, abort the others
        timeout(time: 15, unit: 'MINUTES')   // never let a hung build run forever
    }

    environment {
        CI = 'true'                          // many tools (incl. Vite/ESLint) behave differently in CI
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git --version'
            }
        }

        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Verify') {
            parallel {
                stage('Lint') {
                    steps {
                        // Portfolio is NOT lint-clean yet, so lint is ADVISORY:
                        // the stage shows red, but the overall build stays SUCCESS.
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh 'npm run lint'
                        }
                    }
                }
                stage('Build') {
                    steps {
                        sh 'npm run build'
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Result: ${currentBuild.currentResult} (build #${env.BUILD_NUMBER})"
        }
        success {
            archiveArtifacts artifacts: 'dist/**', fingerprint: true
            script {
                currentBuild.description = "OK • ${env.BUILD_NUMBER} • ${env.GIT_COMMIT?.take(7) ?: 'local'}"
            }
        }
        failure {
            script {
                currentBuild.description = "FAILED • ${env.BUILD_NUMBER}"
            }
        }
        cleanup {
            deleteDir()                      // always clean the workspace, even on failure
        }
    }
}
```

**Explain:**
- The two parallel branches **don't declare their own `agent`**, so they inherit the top-level agent and **share the
  same workspace** — that's why `Lint` and `Build` can both see the `node_modules` installed in the `Install` stage.
  (If you gave each branch its own `agent`, Jenkins would allocate a *separate* workspace and you'd have to reinstall.)
- `parallelsAlwaysFailFast()` + 2 executors on the node = genuine parallel execution with prompt cancellation.
- `catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE')` makes lint an **advisory gate**: you still *see* the red
  `Lint` stage, but it does **not** fail the build. This is handy when a project isn't lint-clean yet; once your project
  passes lint, delete the `catchError` wrapper to turn it into a hard gate.
- `cleanup { deleteDir() }` runs last, no matter what — the next build always starts clean.
- We archive artifacts only on `success`.

Commit and push:

```bash
cd my-portfolio
git add Jenkinsfile
git commit -m "ci: add lint/build quality gates and structure"
git push origin main
```

Then in Jenkins click **Build Now**. (There is no automatic trigger yet — that comes in Phase 4 with the GitHub
webhook / SCM polling. For now, every build is started manually.)

---

## Step 2 — Custom agent image with extra tools (optional but recommended)

**Why:** the stock inbound-agent is minimal. Pre-baking small CLI tools avoids installing them on every run and
demonstrates how production agents are built.

Create `jenkins-learning/jenkins/Dockerfile`:

```dockerfile
# jenkins/Dockerfile
FROM jenkins/inbound-agent:latest-jdk21

USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      git \
      curl \
      jq \
      ca-certificates \
 && rm -rf /var/lib/apt/lists/*
USER jenkins
```

Update the `agent` service in `docker-compose.yml` to build it instead of pulling the base image:

```yaml
  agent:
    build:
      context: ./jenkins
      dockerfile: Dockerfile
    image: my-portfolio-jenkins-agent:latest        # name the built image
    container_name: jenkins-agent
    restart: unless-stopped
    user: root
    environment:
      - JENKINS_URL=http://controller:8080
      - JENKINS_AGENT_NAME=docker-agent
      - JENKINS_SECRET=${JENKINS_AGENT_SECRET}
      - JENKINS_AGENT_WORKDIR=/home/jenkins/agent
    volumes:
      - agent_workspace:/home/jenkins/agent
    depends_on:
      - controller
    networks:
      - jenkins-net
```

Rebuild and restart the agent:

```bash
cd jenkins-learning
docker compose up -d --build agent
docker compose logs -f agent
```

> Gotcha: rebuilding recreates the container. The node is already registered, so the same `JENKINS_AGENT_SECRET`
> in `.env` still pairs it — you only need a new secret if you delete/recreate the **node** in the UI.

---

## Test — verify Phase 3 works

1. **Build Now** → both `Lint` and `Build` appear side-by-side in the **Pipeline Stage View**, with timestamps in the
   console output. ✅
2. **Observe the advisory lint gate:** if your project isn't lint-clean yet, `Lint` shows **red** while the overall
   build stays **SUCCESS** and `dist/` is still archived — that's `catchError` working as intended. If your project
   already passes lint, this stage will simply be green. ✅
3. **Force a real failure:** break the build on purpose (e.g. introduce a syntax error in a `.jsx` file), commit + push.
   - The `Build` branch goes red, the build result is **FAILURE**, and **no** artifact is archived. ✅
   - The other branch is aborted promptly (fail-fast). ✅
   - Revert the error afterward.
4. **Retention:** after several builds, only the 10 most recent are kept (visible in Build History). ✅
5. Console shows `Result: SUCCESS` in the `always` post block. ✅

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Lint` stage red but build green | Advisory `catchError` working as designed | Remove `catchError` once the repo is lint-clean |
| `deleteDir` wiped node_modules, next stage fails | Expected — `cleanup` is last, so it's fine | n/a |
| Parallel branches "not truly parallel" | Node has only 1 executor | Set executors to 2 in the node config (Phase 1) |
| `timestamps` unknown step | Timestamper plugin missing | Install it (Phase 1 Step 4) |
| Build hangs at 15 min | Something didn't exit | Check the last console lines; `timeout` auto-aborts |

**Next:** Phase 4 wires GitHub to Jenkins with credentials and a webhook so pushes trigger builds automatically.

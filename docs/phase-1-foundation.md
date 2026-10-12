# Phase 1 — Foundation: Jenkins up in Docker

## Goal
Stand up a Jenkins **controller** and a dedicated **inbound agent** with Docker Compose, complete the setup wizard,
install the plugins we need, create your admin user, and confirm the agent is **online**.

**End state:** `http://localhost:8080` shows Jenkins, and one agent named `docker-agent` is connected and ready.

---

## Concepts

| Concept | Why it matters here |
|---------|---------------------|
| **Controller** | The brain: UI, job scheduling, config, history. Runs in its own container. |
| **Inbound agent** | A worker that dials into the controller using a secret. Isolated from the controller. |
| **`JENKINS_HOME`** | Where all Jenkins state lives. We mount it as a **named volume** so it survives restarts. |
| **Docker Compose** | Declares both containers + network + volumes in one file. |
| **Agent secret** | A controller-generated credential for one inbound-agent node. The agent reuses it to authenticate until that node is recreated or its secret changes. |

---

## Dependencies
None. This is the starting point.

---

## Step 1 — Create the workspace files

Create `docker-compose.yml` in `jenkins-learning/`:

```yaml
# docker-compose.yml
services:
  controller:
    image: jenkins/jenkins:lts-jdk21          # LTS line, JDK 21
    container_name: jenkins-controller
    restart: unless-stopped
    user: root                                 # simplifies volume permissions for local learning
    ports:
      - "8080:8080"                            # web UI
      - "50000:50000"                          # agent TCP port (used by SSH/other agents)
    volumes:
      - jenkins_home:/var/jenkins_home         # persist all Jenkins state
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=true
    networks:
      - jenkins-net

  agent:
    image: jenkins/inbound-agent:latest-jdk21  # worker image (JDK 21)
    container_name: jenkins-agent
    restart: unless-stopped
    user: root                                 # allow tool installs + workspace writes in the container
    environment:
      - JENKINS_URL=http://controller:8080     # reach the controller over the compose network
      - JENKINS_AGENT_NAME=docker-agent        # must match the node name you create in the UI
      - JENKINS_SECRET=${JENKINS_AGENT_SECRET} # filled from .env AFTER creating the node
      - JENKINS_AGENT_WORKDIR=/home/jenkins/agent
    volumes:
      - agent_workspace:/home/jenkins/agent    # keep workspace + installed tools across restarts
    depends_on:
      - controller
    networks:
      - jenkins-net

volumes:
  jenkins_home:
  agent_workspace:

networks:
  jenkins-net:
```

**Why `user: root`?** The official images run as an unprivileged `jenkins` user, which fights with host-mounted
volume permissions. For a local learning box this is the pragmatic choice; in production you'd fix ownership instead.
We use **named volumes** (not bind mounts) precisely so root vs. user doesn't bite you.

**Why two images?** The controller image is the server; the inbound-agent image ships a JNLP client that connects back.

Create `.env.example` (committed) and `.env` (git-ignored, holds the real secret):

```bash
# .env.example
JENKINS_AGENT_SECRET=replace-with-secret-from-jenkins-node-page
```

```bash
# .env  (create this now; you will paste the real secret in Step 5)
JENKINS_AGENT_SECRET=placeholder-change-me
```

Create `.gitignore`:

```gitignore
.env
```

Create `docs/` (already exists) — nothing to do.

**Explain:** We keep the agent secret out of Git. `.env.example` documents the variable; `.env` holds the value.

---

## Step 2 — Start the controller only

```bash
docker compose up -d controller
```

Wait for it to boot, then confirm:

```bash
docker compose logs -f controller
```

You'll see `Jenkins is fully up and running`. Press `Ctrl+C` to stop following logs.

**Test:** open `http://localhost:8080` — you should see the "Unlock Jenkins" page.

---

## Step 3 — Unlock Jenkins

Get the initial admin password:

```bash
docker exec jenkins-controller cat /var/jenkins_home/secrets/initialAdminPassword
```

Paste it into the unlock screen. Then:
1. Choose **Install suggested plugins** and wait for installation.
2. Create your first **admin user** (remember the credentials).
3. Keep the default URL (`http://localhost:8080/`) and finish.

---

## Step 4 — Install the extra plugins we need

**Why:** the suggested set does not include the Node.js tool manager.

Go to **Manage Jenkins → Plugins → Available plugins**, search and install:

| Plugin | Purpose |
|--------|---------|
| **NodeJS** | Lets Jenkins download and manage a Node.js runtime (Phase 2). |
| **Pipeline: Stage View** | Visual stage grid on the job page (nice to have). |
| **GitHub** | GitHub webhook + commit status integration (Phase 4). Usually already installed (see note). |
| **Timestamper** | Adds timestamps to console output (Phase 3). Usually already installed (see note). |

> **Note — plugins that look "missing":** the **Available plugins** tab only lists plugins that are *not yet
> installed*. Two plugins above are normally pulled in automatically by the suggested set (as direct installs or as
> dependencies), so they will **not** show up there:
> - **GitHub** (`github`) — installed as a dependency of **GitHub Branch Source** (part of suggested plugins).
>   Searching "github" in Available only shows *other* GitHub-related plugins (Branch Source, API, Authentication…).
> - **Timestamper** (`timestamper`) — part of the suggested plugins set; provides the `timestamps()` step used in Phase 3.
>   **Do not** confuse it with **"Secondary Timestamper"**, which is a different, optional plugin we don't need.
>
> Verify both under **Manage Jenkins → Plugins → Installed plugins** (search `github` and `timestamper`). Only install
> from Available if they are genuinely absent.

Check **Restart Jenkins when installation is complete** (or restart the container later):
```bash
docker compose restart controller
```

---

## Step 5 — Create the agent node and pair the container

1. **Manage Jenkins → Nodes → + New Node**.
2. **Node name:** `docker-agent` (must match `JENKINS_AGENT_NAME`).
3. Select **Permanent Agent** → **Create**.
4. Fill in:
   - **Number of executors:** `2` (Phase 3 uses parallel stages).
   - **Remote root directory:** `/home/jenkins/agent`
   - **Labels:** `docker linux`
   - **Usage:** "Use this node as much as possible"
   - **Launch method:** **Launch agent by connecting it to the controller**
5. **Save**. You'll land on the node's status page — click the node name to open it.
6. Copy the **Secret** value shown. This secret is for this node (not a global controller secret); the agent sends it when connecting so the controller can authenticate it.

Paste that secret into your `.env`:

```bash
# .env
JENKINS_AGENT_SECRET=<paste-the-secret-here>
```

Now start the agent:

```bash
docker compose up -d agent
docker compose logs -f agent
```

You should see `Connected` / `Agent successfully connected`.

---

## Test — verify Phase 1 works

1. **Manage Jenkins → Nodes** shows `docker-agent` with a **green** check and 2 executors. ✅
2. The agent logs contain `Connected to http://controller:8080` and no repeated reconnection loop. ✅
3. Run this to prove the agent is talking to the controller:
   ```bash
   docker compose ps
   ```
   Both `jenkins-controller` and `jenkins-agent` show `Up`. ✅

If the agent shows as offline with "invalid secret": re-copy the secret from the node's page (secrets rotate if the node
is recreated), update `.env`, then `docker compose up -d agent` again.

**Next:** Phase 2 adds a `Jenkinsfile` to `my-portfolio` and runs its first build on this agent.

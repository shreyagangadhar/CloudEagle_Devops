## 1. Branching Strategy

| Branch | Environment | What happens |
|---|---|---|
| `feature/*`, `hotfix/*` | — | Build + Test + SonarCloud only (PR validation) |
| `develop` | QA | Above + deploy to QA VM |
| `release/*` | Staging | Above + deploy to Staging VMs |
| `main` | Production | Above + **manual approval** + Blue/Green deploy |

**How accidental prod deploys are prevented:**

- Only `main` triggers a prod deploy — no wildcard matching.
- Before any prod deploy, Jenkins pauses and waits for a member of the `release-managers` group to click **Deploy**. The gate **auto-aborts after 30 minutes** if no one approves — the pipeline fails rather than hanging indefinitely.
- Feature and hotfix branches never upload an artifact or touch any environment.

---

## 2. Jenkins Pipeline

### Stages (in order)

```
Checkout → Build → Test → SonarCloud → Quality Gate
    → [PR stops here]
    → Upload Artifact → [Approve Prod?] → Deploy → Health Check
```

| Stage | What it does |
|---|---|
| **Checkout** | Clones repo, sets artifact name as `appname-gitsha-buildnumber.jar` for full traceability |
| **Build** | `mvn clean package` — skips tests for speed |
| **Test** | Runs unit tests, generates JaCoCo coverage report. JUnit results are **always published to Jenkins UI** even if tests fail, so individual failures are visible in the build view |
| **SonarCloud** | Sends coverage, bytecode, and **branch name** to SonarCloud — analysis runs per-branch, not just on main |
| **Quality Gate** | Waits up to 5 min; **fails the pipeline** if quality gate doesn't pass |
| **Upload Artifact** | Renames the Maven JAR to the SHA-based name, then uploads to the environment-specific GCS folder |
| **Approve Prod Deploy** | Manual input step — only runs on `main`, only `release-managers` can approve |
| **Deploy** | Snapshots the current version, then runs rolling (QA/Staging) or blue/green (Prod) |
| **Health Check** | Hits `/actuator/health` on each VM — parses the JSON response (`status == UP`) using Python. Retries every 5s, 18 attempts = 90s total timeout |

### Artifact naming

Every JAR is named `sync-service-<git-sha>-<build-number>.jar`. This makes every artifact uniquely traceable — you can always tell exactly which commit and which Jenkins run produced it. GCS stores all versions, so rollback is just re-deploying a previous file.

### PR vs Merge behaviour

- **PR (feature/hotfix branch):** Pipeline runs Checkout → Build → Test → SonarCloud → Quality Gate. Stops there. No artifact, no deploy.
- **Merge to develop/release/main:** Full pipeline including artifact upload and deploy.

### Rollback Strategy

**Before every deploy**, Jenkins reads `current-version.txt` from GCS and saves it as `PREVIOUS_VERSION`. This is the safety snapshot — it records what is running *right now*, before anything changes.

If the pipeline fails **after the Deploy stage begins**, the `post { failure }` block runs automatically:

- **QA / Staging (Rolling):** Re-downloads the previous JAR from GCS and restarts the service on each VM. The existing `secrets.env` on the VM is reused — secrets are not re-fetched.
- **Prod (Blue/Green):** Flips the GCP load balancer back to the previously live color. No JAR re-copy needed — the old VMs were never stopped.

> Note: Rollback only triggers if both `DEPLOY_ENV != none` and `PREVIOUS_VERSION != none`. If the pipeline fails earlier (e.g. Quality Gate), no rollback runs — nothing was deployed yet.

### Workspace cleanup

After every run (success or failure), `cleanWs()` wipes the Jenkins workspace. This prevents stale artifacts, leftover credential files, or old scripts from interfering with the next build.

---

## 3. Configuration Management

### Environment-specific config

Non-secret configuration (app name, port, GCP project) flows through **pipeline environment variables** defined at the top of the Jenkinsfile. No per-environment config files are needed.

Each deploy target gets its own isolated resources by naming convention:

| Resource | Pattern |
|---|---|
| GCS folder | `gs://bucket/qa/`, `gs://bucket/staging/`, `gs://bucket/prod/` |
| VM tag | `sync-service-qa`, `sync-service-staging`, `sync-service-prod-blue/green` |
| Secret name | `sync-service-qa-mongodb-uri`, `sync-service-prod-mongodb-uri`, etc. |

The `resolveEnv()` function maps branch → env at runtime. Switching environments is just a branch merge — no Jenkinsfile edits needed.

### Dynamic VM discovery

VMs are never hardcoded. The `getVMsByTag()` function queries GCP at deploy time for all **RUNNING** VMs with the matching tag. This means you can scale VMs up or down in GCP and the pipeline automatically picks them up — no Jenkins config changes required.

### Jenkins credentials (two separate ones)

| Credential ID | Type | Used for |
|---|---|---|
| `gcp-sa-key` | Secret File (JSON key) | `gcloud auth` — lets Jenkins run `gcloud` CLI commands |
| `gcp-gcs-credentials` | GCS Plugin credential | Jenkins GCS plugin for uploading/downloading artifacts |

### Secrets handling

MongoDB credentials and other app secrets are **never stored in the repo, Jenkins, or env vars on the master**. The flow is:

1. `deploy.sh` is written locally, `scp`'d to the VM, and executed there — this avoids shell-quoting issues with multi-step remote commands and keeps the logic readable.
2. The VM's attached service account calls `gcloud secrets versions access` to fetch the MongoDB URI from **GCP Secret Manager**. Note: `gcloud storage cp` is used throughout (the current recommended replacement for `gsutil cp`).
3. The value is written to `/etc/sync-service/secrets.env` with `chmod 600` — never echoed to logs.
4. The file is owned by `sync-service:sync-service` — the service runs as a dedicated OS user, not root.
5. Spring Boot reads it as an environment variable at startup via systemd.

### Shell script safety (`set -euo pipefail`)

All remote scripts (`deploy.sh`, `rollback.sh`, `healthcheck.sh`) start with `set -euo pipefail` — any failed command aborts the script immediately rather than silently continuing with a bad state.

---

## 4. Deployment Strategy

| Environment | Strategy | Why |
|---|---|---|
| QA | **Rolling** | Low risk, simple |
| Staging | **Rolling** | Low risk, simulates prod-like behaviour, cheap |
| Prod | **Blue/Green** | Zero downtime; near-instant rollback by flipping LB |

### How Rolling works (QA / Staging)

Each VM is deployed and health-checked **one at a time** before moving to the next. If a VM fails its health check, the pipeline stops — remaining VMs are still on the old version, so the service keeps running in a degraded but functional state.

### How Blue/Green works (Prod)

Two sets of VMs exist: `sync-service-prod-blue` and `sync-service-prod-green`. One is live, one is idle.

**Deploy flow:**
1. Jenkins reads `live-color.txt` from GCS to find which color is live. If the file doesn't exist (first ever prod deploy), it defaults to `blue`.
2. New code is deployed to the **idle** color VMs only.
3. Health checks run directly against the idle VMs (not through the load balancer).
4. If healthy, the GCP backend service is updated:
   - New color → `capacity-scaler=1.0` (receives 100% of traffic)
   - Old color → `capacity-scaler=0.0` (receives 0% of traffic, VMs stay running)
5. `live-color.txt` in GCS is updated to the new live color.

**`capacity-scaler` is how traffic switching works** — flipping these values shifts all traffic instantly with no VM restarts.

**Rollback:** Flip the `capacity-scaler` values back. Takes effect in seconds, no redeploy needed.

This gives **zero-downtime deployments** and **near-instant rollback** on prod.



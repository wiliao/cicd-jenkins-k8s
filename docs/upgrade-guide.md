# Upgrade Guide: cicd-jenkins-k8s

This guide walks through closing the gaps identified in the repo review, in priority order. Each section includes what to change, why, and the concrete steps/snippets to get there.

---

## 1. Add pipeline-as-code (Jenkinsfile)

**Problem:** The build is configured as a Jenkins Freestyle project through the UI. None of that configuration is checked into the repo, so the pipeline can't be reproduced, reviewed, or versioned.

**Fix:**
1. In the `mytest` Spring Boot project (or a new `pipeline/` folder in this repo if you want to keep a reference copy), add a `Jenkinsfile`:

   ```groovy
   pipeline {
       agent any
       tools {
           maven 'Maven-3'   // configure a Maven tool in Jenkins Global Tool Config
       }
       stages {
           stage('Checkout') {
               steps {
                   checkout scm
               }
           }
           stage('Build') {
               steps {
                   sh 'mvn -B clean package'
               }
           }
           stage('Test') {
               steps {
                   sh 'mvn -B test'
               }
               post {
                   always {
                       junit '**/target/surefire-reports/*.xml'
                   }
               }
           }
           stage('Docker Build') {
               steps {
                   sh 'docker build -t mytest:${BUILD_NUMBER} .'
               }
           }
       }
   }
   ```

2. In Jenkins, convert the Freestyle job to a **Pipeline** job pointing at this `Jenkinsfile` (Pipeline script from SCM → your GitLab repo → `Jenkinsfile` path).
3. Delete the old Freestyle job once the Pipeline job is verified working, so there's a single source of truth.
4. Commit the `Jenkinsfile` alongside the application code so build/test/deploy steps are versioned with the code they build.

---

## 2. Add Kubernetes manifests (or rename the repo)

**Problem:** The repo is named `cicd-jenkins-k8s` but contains no Kubernetes manifests — only Docker Desktop/WSL2 containers.

**Fix — pick one:**

**Option A: Add real K8s deployment**
1. Create a `k8s/` directory with a basic Deployment + Service for `mytest`:

   ```yaml
   # k8s/deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mytest
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: mytest
     template:
       metadata:
         labels:
           app: mytest
       spec:
         containers:
           - name: mytest
             image: mytest:latest
             ports:
               - containerPort: 8080
   ```

   ```yaml
   # k8s/service.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: mytest
   spec:
     selector:
       app: mytest
     ports:
       - port: 80
         targetPort: 8080
     type: ClusterIP
   ```

2. Add a `Deploy` stage to the Jenkinsfile from Step 1:

   ```groovy
   stage('Deploy to K8s') {
       steps {
           sh 'kubectl apply -f k8s/'
       }
   }
   ```

3. Enable Kubernetes in Docker Desktop (Settings → Kubernetes → Enable Kubernetes) so `kubectl apply` has a local cluster to target.

**Option B: Rename the repo**
- If Kubernetes isn't actually part of the scope, rename to something accurate, e.g. `cicd-jenkins-gitlab-docker`, and update the README title/description to match. Simpler, but loses the K8s learning angle if that was the goal.

---

## 3. Remove hardcoded IP from GitLab Compose file

**Problem:** `gitlab/docker-compose-gitlab.yml` hardcodes a WSL2 IP (`172.22.216.211`) that changes on reboot.

**Fix:**
1. Introduce a `.env` file (gitignored) with:
   ```
   GITLAB_EXTERNAL_URL=http://localhost:8929
   ```
2. Reference it in the Compose file:
   ```yaml
   services:
     gitlab:
       image: 'gitlab/gitlab-ce:16.9.2-ce.0'
       environment:
         GITLAB_OMNIBUS_CONFIG: |
           external_url '${GITLAB_EXTERNAL_URL}'
           gitlab_rails['gitlab_shell_ssh_port'] = 2224
   ```
3. Add a `.env.example` with the same key and a placeholder value, so others know what to set without needing your actual WSL2 IP.
4. Document in the README that `localhost` works fine for same-machine access; the WSL2 IP is only needed if accessing from another device on the network — and even then, use `hostname -I` to fetch it fresh rather than hardcoding.

---

## 4. Pin the GitLab image version

**Problem:** `gitlab/gitlab-ce:latest` isn't reproducible — pulling it next month may give you a different version than what was tested.

**Fix:**
1. Pick a specific tag, e.g. `gitlab/gitlab-ce:16.9.2-ce.0` (check [GitLab's Docker Hub tags](https://hub.docker.com/r/gitlab/gitlab-ce/tags) for the current stable release).
2. Update `docker-compose-gitlab.yml`:
   ```yaml
   image: 'gitlab/gitlab-ce:16.9.2-ce.0'
   ```
3. Document the tested version in the README so future readers know which version the walkthrough screenshots correspond to.

---

## 5. Document credential handling

**Problem:** The README shows initial passwords being generated but doesn't explain how credentials should be managed afterward (Jenkins credentials store, GitLab tokens, SSH keys for "Publish over SSH").

**Fix:**
1. Add a `SECURITY.md` or a "Credentials" section in the README covering:
   - Rotate the GitLab root password and Jenkins admin password immediately after initial setup (already implied — make it explicit and mandatory).
   - Store the GitLab personal access token (used by the "Publish Over SSH" / GitLab webhook integration) in **Jenkins → Manage Jenkins → Credentials**, never in the Jenkinsfile or Compose file.
   - Generate an SSH key pair for Jenkins → GitLab access, store the private key in Jenkins Credentials (SSH Username with private key), and add the public key to the GitLab deploy keys.
2. Add a `.env.example` (see Step 3) so real secrets never get committed, and add `.env` to `.gitignore` (see Step 6).

---

## 6. Add a `.gitignore`

**Problem:** No `.gitignore` exists; as the repo grows to include real project code or `.env` files, secrets or build artifacts could get committed accidentally.

**Fix:** Add a `.gitignore` at the repo root:

```gitignore
# Environment / secrets
.env
*.pem
*.key

# GitLab local data (if running docker-compose in-repo)
gitlab/config/
gitlab/data/
gitlab/logs/

# Build artifacts
target/
*.class
*.jar

# OS/editor cruft
.DS_Store
.idea/
*.iml
```

---

## 7. Add CI for the repo itself

**Problem:** A "CI/CD" repo has no CI of its own to validate the Dockerfile and Compose file.

**Fix:** Add `.github/workflows/lint.yml`:

```yaml
name: Lint infra files

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: jenkins/Dockerfile

      - name: Validate docker-compose
        run: docker compose -f gitlab/docker-compose-gitlab.yml config
```

This catches Dockerfile best-practice issues (via hadolint) and Compose syntax errors (via `docker compose config`) on every push/PR.

---

## Suggested order of work

1. `.gitignore` (5 min, prevents future mistakes)
2. Pin GitLab image + remove hardcoded IP (quick, low risk)
3. Document credential handling (no code changes, just docs)
4. Add CI lint workflow (quick, catches regressions going forward)
5. Add Jenkinsfile (bigger lift, but the highest-value fix)
6. Add K8s manifests or rename the repo (decide direction first, then execute)

Once these are in place, re-run through the README's 11 steps end-to-end to confirm the walkthrough still matches reality, and update any screenshots that changed.

# Day 3 — Git & Jenkins (Production-Level DevOps)

**Duration:** 2 Hours | **Focus:** How code moves from a developer's laptop to production

This folder documents Day 3 of the DevOps prep plan: advanced Git, Jenkins pipelines, production troubleshooting, and interview-ready answers.

---

## 📁 Contents

| File | Covers |
|---|---|
| [`Git-Commands.md`](./Git-Commands.md) | Git internals (working dir/staging/local/remote), branching, merging, conflict resolution, stash, log, revert, reset, rebase, cherry-pick, tags |
| [`Jenkins-Pipeline.md`](./Jenkins-Pipeline.md) | Freestyle vs Pipeline jobs, Jenkinsfile syntax, triggering builds (webhook/polling), reading console output, full CI/CD flow |
| [`Jenkins-Troubleshooting.md`](./Jenkins-Troubleshooting.md) | Step-by-step process for diagnosing a failed Jenkins build in production |

---

## ✅ Task Checklist

### Task 1 — Advanced Git (45 min)
- [x] Initialize a Git repository
- [x] Create a new branch
- [x] Merge branches
- [x] Resolve a merge conflict
- [x] Use `git stash`
- [x] View commit history with `git log`
- [x] Revert a commit
- [x] Reset a commit (soft vs hard)
- [x] Push code to GitHub

### Task 2 — Jenkins Pipeline (45 min)
- [x] Create a Freestyle Job
- [x] Create a Pipeline Job
- [x] Understand a Jenkinsfile
- [x] Configure GitHub webhook or SCM polling
- [x] Trigger a build manually
- [x] Read the Console Output
- [x] Identify where a build failed

### Task 3 — Production Troubleshooting (20 min)
- [x] Check Console Output
- [x] Identify which stage failed
- [x] Verify Git checkout
- [x] Verify credentials
- [x] Verify environment variables
- [x] Verify disk space
- [x] Verify tool versions (Java, Maven, Node, etc.)
- [x] Check Jenkins logs

### Task 4 — GitHub Documentation (10 min)
- [x] Create `Day-3` folder
- [x] Add `Git-Commands.md`
- [x] Add `Jenkins-Pipeline.md`
- [x] Add `Jenkins-Troubleshooting.md`
- [x] Commit and push to GitHub

---

## 🔄 CI/CD Flow (Code Commit → Deployment)

```
Developer
    ↓
Git Push
    ↓
GitHub
    ↓
Jenkins
    ↓
Build
    ↓
Unit Tests
    ↓
SonarQube
    ↓
Docker Image
    ↓
Docker Registry
```

---

## 🎯 Interview Questions & Answers

### 1. What happens after you run `git push`?
Git takes the commits that exist in your local branch but not yet on the remote-tracking branch, and transfers them to the remote repository (e.g., GitHub) over HTTPS or SSH. Specifically:
1. Git determines which commits are new locally (not on the remote).
2. It packs those commits' objects (blobs, trees, commits) and sends them to the remote.
3. The remote updates its branch reference (e.g., `refs/heads/main`) to point to your new commit.
4. If a **webhook** is configured (as in our Jenkins setup), GitHub immediately notifies Jenkins, which can automatically trigger a CI/CD pipeline.
5. If someone else pushed in the meantime and your branch is behind, the push is **rejected** ("non-fast-forward") — you'd need to `git pull`/rebase first.

### 2. Difference between Merge and Rebase?
**Merge** combines two branches by creating a new merge commit with two parents — it preserves full, non-linear history exactly as it happened. **Rebase** takes your branch's commits and replays them one by one on top of the target branch, producing a clean, linear history — but it rewrites commit hashes, which makes it unsafe for branches others are already working from. Rule of thumb: rebase private/local branches to clean up history before opening a PR; merge when integrating into shared branches like `main`.

### 3. What is a Jenkinsfile?
A Jenkinsfile is a text file, checked into the project's Git repository, that defines the entire CI/CD pipeline as code using Groovy-based syntax (Declarative or Scripted). It specifies stages like checkout, build, test, code analysis, image build, and deployment. Because it lives in version control, pipeline changes go through the same review process as application code — this is the "Pipeline as Code" principle that makes Jenkins pipelines reproducible and auditable.

### 4. Difference between Freestyle and Pipeline jobs?
A **Freestyle job** is configured manually through the Jenkins UI — simple to set up but not version-controlled and hard to scale for multi-stage workflows. A **Pipeline job** is defined in code (a Jenkinsfile) stored in the repo, supports complex multi-stage logic, parallel execution, and is reviewable/reusable like any other code. Production environments favor Pipeline jobs because the pipeline definition travels with the codebase and is auditable.

### 5. Why would a Jenkins build suddenly fail if it worked yesterday?
Almost always an **environment change**, not a code change, since no code changed. The usual suspects, in order of likelihood: an expired credential/token (GitHub PAT, registry login), an unpinned dependency or tool auto-updating to a new version overnight (Java/Node/Maven), the Jenkins agent running out of disk space (common with Docker image builds), a plugin auto-update changing behavior, or an external service (SonarQube, artifact registry) being briefly down. The fix is to check what changed in the environment between the last successful build and this one.

### 6. What information do you check first when a pipeline fails?
The **Console Output** of the failed build — it's the single source of truth for what happened. From there, identify exactly which **stage** failed (Checkout, Build, Test, SonarQube, Docker Build, Push), since that immediately narrows the likely cause category (Git/credentials issue vs. code issue vs. infrastructure issue). Only after reading the console output and pinpointing the stage do you move to targeted checks — credentials, environment variables, disk space, or tool versions.

---

## 🎯 Daily Goal — Self-Check

By the end of Day 3, I can:
- [x] Use Git confidently in a team environment
- [x] Explain a CI/CD pipeline from code commit to deployment
- [x] Troubleshoot common Jenkins pipeline failures
- [x] Document my learning in GitHub


# Git Commands & Concepts — Day 3 Notes

## 1. The Four Areas of Git

Understanding these four areas is the foundation of everything else in Git.

| Area | What it is | Command that moves code into it |
|---|---|---|
| **Working Directory** | The actual files on your disk that you edit | (you editing files) |
| **Staging Area (Index)** | A holding area for changes you want to include in the next commit | `git add <file>` |
| **Local Repository** | The `.git` folder on your machine — a full history of commits | `git commit -m "message"` |
| **Remote Repository** | The copy of the repo on GitHub/GitLab/Bitbucket | `git push` |

**Flow:**
```
Working Directory --(git add)--> Staging Area --(git commit)--> Local Repo --(git push)--> Remote Repo
```

This separation is what lets you selectively stage only some changes, review before committing, and commit locally before ever touching the network.

---

## 2. Core Commands Practiced

### Initialize a repository
```bash
git init
```
Creates a hidden `.git` folder — this is what turns a normal folder into a Git repository. No history exists yet; this just sets up tracking.

### Create a new branch
```bash
git branch feature-login       # create
git checkout feature-login     # switch
# or in one step:
git checkout -b feature-login
# modern syntax:
git switch -c feature-login
```
A branch is just a movable pointer to a commit. Creating one is cheap and instant — it doesn't copy files.

### Merge branches
```bash
git checkout main
git merge feature-login
```
Combines the history of `feature-login` into `main`. If the branches diverged (commits on both sides), Git creates a **merge commit** with two parents.

### Resolve a merge conflict
1. Git marks conflicting sections in the file:
   ```
   <<<<<<< HEAD
   code from current branch
   =======
   code from the branch being merged
   >>>>>>> feature-login
   ```
2. Manually edit the file to keep the correct code and remove the conflict markers.
3. Mark it resolved and finish the merge:
   ```bash
   git add <file>
   git commit
   ```

### Stash changes
```bash
git stash              # save uncommitted changes, restore clean working dir
git stash list          # see all stashes
git stash pop           # reapply the latest stash and remove it from the list
git stash apply         # reapply without removing from the list
```
Useful when you need to switch branches urgently (e.g., a production hotfix) but aren't ready to commit your current work.

### View commit history
```bash
git log                     # full history
git log --oneline           # compact, one line per commit
git log --oneline --graph --all   # visual branch/merge structure
```

### Revert a commit
```bash
git revert <commit-hash>
```
Creates a **new commit** that undoes the changes of a previous commit. History is preserved — this is the **safe** way to undo something that has already been pushed/shared.

### Reset a commit — soft vs hard
```bash
git reset --soft <commit-hash>   # moves HEAD, keeps changes staged
git reset --mixed <commit-hash>  # moves HEAD, keeps changes unstaged (default)
git reset --hard <commit-hash>   # moves HEAD, DISCARDS all changes
```

| Type | Working Directory | Staging Area | Commit History |
|---|---|---|---|
| `--soft` | Unchanged | Unchanged | Moved back |
| `--mixed` | Unchanged | Reset | Moved back |
| `--hard` | **Reset (data lost)** | Reset | Moved back |

⚠️ `reset --hard` rewrites history — never use it on commits that have already been pushed and shared with others. Use `revert` instead for shared branches.

### Push code to GitHub
```bash
git remote add origin <repo-url>   # link local repo to remote (first time only)
git push -u origin main            # push and set upstream tracking
git push                           # subsequent pushes
```

---

## 3. Merge vs Rebase

Both integrate changes from one branch into another, but they do it differently.

| | Merge | Rebase |
|---|---|---|
| **History** | Preserves exact history, adds a merge commit | Rewrites history — replays your commits on top of the target branch |
| **Result** | Non-linear, "true" history with branch points visible | Linear, clean history — looks like work happened sequentially |
| **Safety on shared branches** | Safe — doesn't rewrite existing commits | Unsafe on shared/public branches — rewrites commit hashes |
| **Command** | `git merge <branch>` | `git rebase <branch>` |
| **Typical use** | Merging feature branches into `main` | Cleaning up your local feature branch before opening a PR |

**Rule of thumb:** Rebase local/private branches to keep history clean. Merge when integrating into shared branches like `main` or `develop`.

---

## 4. Cherry-pick

```bash
git cherry-pick <commit-hash>
```
Applies a **single specific commit** from one branch onto another, without merging the entire branch. Common use case: you need one bug-fix commit from a feature branch onto `main`, without pulling in the rest of the unfinished feature.

---

## 5. Tags

Tags mark specific points in history — typically used for releases/versions.

```bash
git tag v1.0.0                          # lightweight tag
git tag -a v1.0.0 -m "Release 1.0.0"    # annotated tag (recommended — has metadata)
git push origin v1.0.0                   # push a single tag
git push origin --tags                   # push all tags
```
Unlike branches, tags don't move — they permanently point to one commit (e.g., "this exact commit is what we deployed as v1.0.0").

---

## 6. Quick Reference Table

| Concept | Command |
|---|---|
| Init repo | `git init` |
| Clone repo | `git clone <url>` |
| Check status | `git status` |
| Stage changes | `git add <file>` / `git add .` |
| Commit | `git commit -m "message"` |
| Create/switch branch | `git checkout -b <branch>` |
| Merge | `git merge <branch>` |
| Rebase | `git rebase <branch>` |
| Stash | `git stash` / `git stash pop` |
| View history | `git log --oneline --graph` |
| Undo (safe) | `git revert <hash>` |
| Undo (destructive) | `git reset --hard <hash>` |
| Cherry-pick | `git cherry-pick <hash>` |
| Tag | `git tag -a v1.0 -m "msg"` |
| Push | `git push origin <branch>` |
| Pull | `git pull origin <branch>` |

# Jenkins Pipeline — Day 3 Notes

## 1. Freestyle Job vs Pipeline Job

| | Freestyle Job | Pipeline Job |
|---|---|---|
| **Configuration** | Configured through the Jenkins UI (point-and-click) | Defined as code in a `Jenkinsfile` |
| **Version control** | Not version-controlled by default; config lives inside Jenkins | Lives in your Git repo, versioned alongside your code |
| **Complexity handling** | Good for simple, single-step jobs | Handles complex multi-stage workflows (build → test → deploy) |
| **Reusability** | Hard to replicate across projects | Easy to copy/reuse as a `Jenkinsfile` template |
| **Review/Audit** | Changes aren't reviewed via PRs | Changes go through code review like any other code |
| **Best for** | Quick one-off tasks, legacy setups | Production-grade CI/CD (industry standard today) |

**In interviews:** Pipeline jobs are considered the modern, production-standard approach because the pipeline definition is "infrastructure as code" — reviewable, versioned, and portable.

---

## 2. What is a Jenkinsfile?

A **Jenkinsfile** is a text file (checked into your Git repository, usually at the project root) that defines the entire CI/CD pipeline as code, using Groovy-based syntax.

There are two styles:

**Declarative (recommended, easier to read):**
```groovy
pipeline {
    agent any

    environment {
        MAVEN_HOME = tool 'Maven3'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/your-org/your-repo.git'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                sh 'mvn sonar:sonar'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
            }
        }
        stage('Push to Registry') {
            steps {
                sh 'docker push myrepo/myapp:${BUILD_NUMBER}'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed — check console output.'
        }
    }
}
```

**Scripted (older, more flexible, pure Groovy):**
```groovy
node {
    stage('Checkout') {
        git 'https://github.com/your-org/your-repo.git'
    }
    stage('Build') {
        sh 'mvn clean compile'
    }
}
```

**Why Jenkinsfile matters in production:**
- It's version-controlled — every change to the pipeline is tracked and reviewable.
- It enables "Pipeline as Code" — the same pipeline can run consistently across environments.
- It supports parallel stages, retries, timeouts, and conditional logic.

---

## 3. Triggering a Build

### Manual trigger
Jenkins Dashboard → select job → **Build Now**.

### GitHub Webhook (event-driven, preferred in production)
1. In GitHub repo: **Settings → Webhooks → Add webhook**
2. Payload URL: `http://<jenkins-url>/github-webhook/`
3. Content type: `application/json`
4. Trigger on: **push events** (and/or PR events)
5. In Jenkins job config: enable **"GitHub hook trigger for GITScm polling"**

Effect: the moment code is pushed to GitHub, GitHub notifies Jenkins instantly, and a build starts automatically. This is the standard production setup — fast feedback, no polling delay.

### SCM Polling (fallback, used when webhooks aren't possible — e.g., Jenkins behind a firewall)
In job config → **Build Triggers → Poll SCM**:
```
H/5 * * * *
```
Jenkins checks the repository every 5 minutes for new commits and triggers a build if changes are found. Less efficient than webhooks (adds delay, adds load) but works when Jenkins can't be reached from the internet.

---

## 4. Reading Console Output

Job → build number (e.g., `#42`) → **Console Output**.

This is the **first place** to look when something goes wrong. It shows:
- Every shell command executed and its output
- Which **stage** the pipeline was in when it failed (declarative pipelines clearly print `[Pipeline] { (Stage Name)}`)
- The exact error message and stack trace
- The final exit code of the failing step

---

## 5. The Full CI/CD Flow

```
Developer  → writes code
    ↓
Git Push   → commits pushed to feature/main branch
    ↓
GitHub     → central remote repo, triggers webhook
    ↓
Jenkins    → picks up the event, starts pipeline
    ↓
Build      → compiles code (e.g., mvn compile / npm build)
    ↓
Unit Tests → runs automated tests (e.g., mvn test / npm test)
    ↓
SonarQube  → static code analysis — bugs, code smells, security issues, coverage
    ↓
Docker Image → application packaged into a container image
    ↓
Docker Registry → image pushed to a registry (Docker Hub / ECR / Nexus / GCR)
    ↓
(Next stage, often) Deploy → image pulled and deployed to staging/production (K8s, ECS, etc.)
```

**Why each stage exists:**
- **Build** — catches compilation errors early.
- **Unit Tests** — catches logic errors before code reaches later stages.
- **SonarQube** — enforces code quality gates; can fail the pipeline if quality thresholds aren't met.
- **Docker Image** — creates a portable, reproducible deployment artifact.
- **Docker Registry** — makes the image available for deployment tooling (Kubernetes, ECS, etc.) to pull from.

This entire chain is what "CI/CD" means in practice: **Continuous Integration** (build + test + quality-check automatically on every push) feeding into **Continuous Delivery/Deployment** (packaging and shipping automatically).

# Jenkins Production Troubleshooting — Day 3 Notes

## Scenario: A Jenkins build fails. What do you do?

This is one of the most common real-world DevOps interview questions. The key is to show a **systematic, top-down process**, not random guessing.

---

## Step-by-Step Troubleshooting Process

### 1. Check the Console Output
Always start here. Job → Build # → **Console Output**.
It tells you the exact command that failed and the error message/stack trace. Never guess before reading this.

### 2. Identify which stage failed
In a declarative pipeline, the console output clearly shows stage boundaries:
```
[Pipeline] { (Build)
...
[Pipeline] { (Unit Tests)
...
ERROR: script returned exit code 1
```
Knowing the exact stage narrows the problem immediately:
- Fails at **Checkout** → Git/credentials issue
- Fails at **Build** → compilation/dependency issue
- Fails at **Test** → code logic issue or flaky test
- Fails at **Docker Build** → Dockerfile or disk space issue
- Fails at **Push to Registry** → auth/network issue

### 3. Verify Git checkout
- Is the branch/tag specified correctly in the `Jenkinsfile` / job config?
- Does the repo URL still resolve? (renamed repo, changed org, etc.)
- Was there a force-push or deleted branch upstream that broke the reference?
```bash
git ls-remote <repo-url>   # sanity check the remote is reachable
```

### 4. Verify credentials
- Has a GitHub token/SSH key expired or been revoked?
- Is the Jenkins **Credentials ID** referenced in the pipeline still valid and correctly scoped?
- Check **Manage Jenkins → Credentials** for expiry warnings.
- Common failure: a personal access token expired (GitHub now requires periodic renewal) — this is a very frequent "it worked yesterday" cause.

### 5. Verify environment variables
- Are required variables (`JAVA_HOME`, `MAVEN_HOME`, API keys, `DOCKER_REGISTRY`, etc.) still set correctly?
- Did someone change a global Jenkins environment variable or a job-level one?
- Check **Manage Jenkins → System** and the `environment {}` block in the Jenkinsfile.

### 6. Verify disk space
Low disk space on the Jenkins agent/master is a very common, easily-missed cause of failure (especially with Docker image builds, which consume a lot of space).
```bash
df -h
docker system df       # check Docker's disk usage specifically
docker system prune -a # clean up unused images/containers (be careful in production)
```

### 7. Verify tool versions (Java, Maven, Node, etc.)
- Did an agent get auto-updated to a new Java/Node/Maven version overnight?
- Does the project require a specific version that's no longer the default?
```bash
java -version
mvn -version
node -v
```
Mismatched tool versions between "yesterday" and "today" is one of the most classic root causes of "it worked yesterday, it doesn't work today" — often due to an unpinned tool version in Jenkins auto-installing an update.

### 8. Check Jenkins logs
If the problem isn't in the job's console output at all (e.g., Jenkins itself seems unhealthy, agents won't connect, jobs won't even start):
- **Manage Jenkins → System Log**
- Server-level log file (typically `/var/log/jenkins/jenkins.log` on Linux installs)
- Check agent connectivity: **Manage Jenkins → Nodes**

---

## Quick Decision Flow

```
Build failed
    │
    ▼
Read Console Output → identify failing stage
    │
    ├── Checkout stage?      → check Git URL / branch / credentials
    ├── Build stage?         → check dependencies, tool versions, code compile errors
    ├── Test stage?          → check test logic, flaky/environment-dependent tests
    ├── SonarQube stage?     → check quality gate thresholds, Sonar server reachability
    ├── Docker Build stage?  → check Dockerfile syntax, disk space
    ├── Push stage?          → check registry auth, network/firewall
    │
    ▼
Still unclear? → Check Jenkins system logs & agent status
```

---

## "Why would a build suddenly fail if it worked yesterday?" — Common Root Causes

1. **Expired credentials/tokens** (GitHub PAT, Docker registry login, SSH key rotation)
2. **A dependency version changed upstream** (e.g., `npm install` pulling a new minor version with a breaking change, since lockfiles weren't committed/respected)
3. **Disk space filled up** on the Jenkins agent (old builds/Docker images not cleaned up)
4. **Tool auto-update** — Jenkins tool installer fetched a newer Java/Node/Maven version overnight
5. **Someone changed the Jenkinsfile or a shared library** without proper testing
6. **Infrastructure change** — network/firewall rule changed, registry moved, DNS issue
7. **Flaky test** — intermittent failure unrelated to actual code change
8. **A plugin auto-updated** and changed behavior
9. **External service downtime** — SonarQube server, artifact repository, or Docker registry was briefly unavailable

**Interview-ready one-liner:** *"A build that worked yesterday and fails today, with no code changes, is almost always an environment problem — not a code problem: expired credentials, an auto-updated dependency/tool, or a full disk are the top three culprits I check first."*

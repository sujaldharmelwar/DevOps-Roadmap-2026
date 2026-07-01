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

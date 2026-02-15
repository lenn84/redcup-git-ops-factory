# 🏭 The Factory: Self-Hosted CI/CD
**A private GitOps engine automating the build-publish-deploy lifecycle.**

## ⚙️ The Pipeline
1. **Trigger:** `git push` events on self-hosted Gitea.
2. **Build:** Ephemeral `act_runner` containers execute Docker builds via privileged sockets.
3. **Registry:** Artifacts pushed to a private, authenticated container registry.
4. **Deploy:** SSH-based rolling updates triggered by semantic version tags.

## 🛠 Tech Stack
- **Engine:** Gitea Actions (GitHub Actions compatible)
- **Runner:** Custom Go-based runners
- **Security:** Organization-level secret injection; strict isolation of build/deploy tokens.
```mermaid
graph LR
    %% The Trigger
    Dev[👨‍💻 Developer] -->|Git Push| Gitea[📦 Gitea Repo]
    
    %% The Build Phase
    Gitea -->|Webhook Trigger| Runner[🤖 Gitea Runner]
    subgraph "LXC 1002: The Factory Floor"
        Runner -->|Spawns| Build_Container[🐳 Ephemeral Build Container]
        Build_Container -->|Docker Build| Artifact[📦 Docker Image]
        Artifact -->|Push (Local LAN)| Registry[🏢 Private Registry :3000]
    end
    
    %% The Deploy Phase
    Registry -->|SSH Pull| Production[🚀 Production Deployment]
```

## 🏛️ Design Philosophy: "Dormant by Design"

Unlike active polling systems, this pipeline follows a strict:

> **Infrastructure provides capability. Repositories provide intent.**

- **The Rule:**  
  The CI runner is always online but remains dormant.  
  Pushing code does nothing unless the repository explicitly contains:`.gitea/workflows/build.yaml`
  

- **The Benefit:**  
Prevents "surprise builds" and ensures resource efficiency on home lab hardware.

---

## 🛡️ Security Engineering: The "Service Account" Strategy"

A major challenge in self-hosted CI is secret management. I implemented an **Organization-Level Secret Injection** strategy to enforce separation of duties.

- **The Problem:**  Developers should not handle:
- Production SSH keys  
- Registry passwords  

- **The Solution:**

- **Admin Role:**  
  Configures global secrets (`TOKEN_GITEA`, `SERVER_SSH_KEY`) at the organization level.

- **Developer Role:**  
  Inherits these secrets inside workflow files:

  ```yaml
  ${{ secrets.TOKEN_GITEA }}
  ```

- **Result:**  
  A zero-trust architecture where developers can deploy code without ever seeing the credentials that authorize deployment.

---

## ⚡ Engineering Challenges & Solutions

*(This section demonstrates real-world troubleshooting beyond tutorial setups.)*

### 1️⃣ The "Runner Amnesia" Loop

- **Challenge:**  
The Gitea Runner failed to reconnect after restarts, attempting to resolve internal Docker DNS names: `http://gitea:3000` which were unreachable during bootstrap.

- **Solution:**  
Engineered a controlled "Hard Reset" protocol:

```bash
docker volume rm runner_data
```
- **Result:** 
  This forced a fresh registration handshake using the correct host IP: 192.168.1.54
  A persistent identity for the build runner.

2️⃣ The Registry Authentication Trap

Challenge:
The pipeline failed with: 401 Unauthorized


The default Gitea {GITHUB_TOKEN} had read-only access and could not push to the Package Registry.

Solution:
Created a dedicated Service Bot account with write:packages scope and injected it as a secure secret.

Result: 
Ephemeral build containers can securely push artifacts to the local registry.

📘 Standard Integration Procedure (SIP)

To standardize onboarding, I developed the RCIP-01 Protocol:

Repository Requirement
Must contain a Dockerfile
(The "How")

Workflow Contract must include:

```bash
.gitea/workflows/build.yaml
```

(The "When")

Landing Zone Control
Production directories (e.g., /data/production/app) must be pre-provisioned by administrators to prevent unmanaged container sprawl.

🛠️ Tech Stack Details

Orchestration:
Docker Compose (Production) + Gitea Actions (Build)

Runner Technology:

{act_runner} executing Docker-in-Docker builds via privileged socket binding:
```bash
/var/run/docker.sock
```

Network Path:
Zero-latency internal transfers.
The runner pushes to the registry via the Docker bridge:

redcup_net


The physical router is never involved.


---




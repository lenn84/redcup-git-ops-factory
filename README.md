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

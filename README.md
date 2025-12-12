```markdown
# GitOps with Argo CD — quick start (single-repo example)

This repo contains a minimal example showing how to implement GitOps using:
- GitHub Actions to build and push a Docker image to GHCR.
- GitHub Actions to update Kubernetes manifests in `manifests/dev`.
- Argo CD to watch `manifests/dev` and deploy into your Kubernetes cluster.

Contents
- app/index.html          -> small static site
- Dockerfile              -> serve the static site via nginx
- manifests/dev/*         -> k8s deployment & service (Argo CD watches this path)
- .github/workflows/*     -> GitHub Actions workflow that builds, pushes, and bumps manifest

Modes supported
1) Single-repo (default in this example)
   - Code and manifests live in same repo.
   - CI builds image, commits manifest update to the same branch -> Argo CD sees change and deploys.

2) Two-repo (recommended for production)
   - App repo and config (GitOps) repo separated.
   - CI creates a PR in the config repo with updated manifests. Reviewers merge to promote.
   - Requires a PAT or bot account with permissions to push/PR to the config repo.

Prereqs
- GitHub repository (this repo).
- A Kubernetes cluster with kubectl configured locally.
- Argo CD installed in the cluster.
- (Optional) For two-repo flow: a Personal Access Token (PAT) with repo write permissions, stored as a secret in the app repo.

Install Argo CD (quick)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Port‑forward the UI for first login:
kubectl -n argocd port-forward svc/argocd-server 8080:443
# Open https://localhost:8080
# Get password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo

How to use this example (single-repo)
1) Push these files into your repository (main branch).
2) In the repo on GitHub, ensure Actions are enabled.
   - For GHCR: pushing images from the same repo uses the repository GITHUB_TOKEN automatically. No extra secrets required.
3) Open Argo CD and create an Application that points to this repo and path `manifests/dev`
   - You can apply manifests/argocd-application.yaml (edit repoURL and OWNER/REPO).
4) Push a commit to main (or create a PR then merge). The workflow:
   - Builds the image and pushes tags: ghcr.io/<owner>/<repo>:<short-sha> and :latest
   - Updates manifests/dev/deployment.yaml to use the short sha and commits it back to the branch
   - Argo CD detects the new commit and syncs the cluster automatically (if automated sync is enabled)

How to use two-repo flow (high level)
- Replace the manifest-updating commit step with a script that clones the config repo using a PAT, updates the manifest, pushes a branch, and opens a PR (use gh CLI or GitHub API).
- Store the PAT in repo secret CONFIG_REPO_PAT and ensure it has repo access.
- Require PR reviews in the config repo to promote to production.

Verify deployment
kubectl get deploy,svc -l app=demo-app
kubectl get pods -l app=demo-app
kubectl logs -l app=demo-app

Security & production tips
- Do not store plaintext secrets in git. Use SealedSecrets / ExternalSecrets or an Argo CD secret plugin.
- Use a bot account or GitHub App for config repo writes, with minimal scopes.
- Protect branches and require PR reviews for config repo merges.
- Add image scanning (Trivy/Cosign) and signing for production images.

If you want, I can:
- Convert the workflow to the two-repo PR flow with a script that uses CONFIG_REPO_PAT.
- Add an Argo CD ApplicationSet/Helm chart example for multi-environment deployments.
- Commit these files into your repo (I can prepare the commit/PR contents).
```

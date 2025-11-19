# Deploy Guide — pencraft GitOps → GKE 

This file explains, step by step, how to deploy the **pencraft** application using GitOps (ArgoCD) to a Google Kubernetes Engine (GKE) cluster. It is written in easy English.

---

## Quick overview

1. We keep Kubernetes manifests and Helm chart in a Git repo called `pencraft-gitops`.
2. CI (GitHub Actions) builds Docker images and updates files in `pencraft-gitops` (the images in `envs/*/values-*.yaml`).
3. ArgoCD watches `pencraft-gitops` and syncs changes to GKE.

---

## What you need (prerequisites)

* A Google Cloud project and a GKE cluster.
* `kubectl` installed and talking to your GKE cluster.
* `helm` installed locally (optional for local testing).
* A GitHub account and the repo `mokadi-suryaprasad/pencraft-gitops` (or your fork).
* GitHub Actions enabled in your app repo.
* A Docker registry (you are using Google Artifact Registry).
* ArgoCD installed in your GKE cluster.

---

## Repo layout (what files should be in `pencraft-gitops`)

```
pencraft-gitops/
├─ charts/pencraft/                # Helm chart for the app
├─ envs/dev/values-dev.yaml        # dev environment values
├─ envs/preprod/values-preprod.yaml
├─ envs/prod/values-prod.yaml
└─ argocd/
   ├─ apps/pencraft-dev.yaml
   ├─ apps/pencraft-preprod.yaml
   ├─ apps/pencraft-prod.yaml
   └─ project.yaml
```

The values files contain `backend.image` and `frontend.image` entries. CI will update these with the new image tags.

---

## Step 1 — Create namespaces in GKE

Run these commands once to create namespaces where the app will run:

```bash
kubectl create namespace pencraft-dev
kubectl create namespace pencraft-preprod
kubectl create namespace pencraft-prod
```

---

## Step 2 — Install ArgoCD in GKE

Install ArgoCD into the `argocd` namespace:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Open the ArgoCD UI locally with port-forwarding:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# open https://localhost:8080 in your browser
```

Log in to ArgoCD using the initial admin password (stored in a secret). Example:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## Step 3 — Connect your Git repo to ArgoCD

In ArgoCD UI go to Settings → Repositories → Connect Repo and add:

* URL: `https://github.com/mokadi-suryaprasad/pencraft-gitops.git` (or your fork)
* Use HTTPS or SSH depending on your preference.

If repo is private, give ArgoCD credentials (username/token or SSH key).

---

## Step 4 — Add the AppProject (optional but recommended)

Apply the `argocd/project.yaml` from the `pencraft-gitops` repo. It restricts which namespaces and repos are allowed. Example:

```bash
kubectl apply -f argocd/project.yaml -n argocd
```

If you skip this step, ArgoCD will use the `default` project.

---

## Step 5 — Add Applications (dev, preprod, prod)

You can create Applications in ArgoCD by applying the YAML files (they are in `argocd/apps/`). For example:

```bash
kubectl apply -f argocd/apps/pencraft-dev.yaml -n argocd
kubectl apply -f argocd/apps/pencraft-preprod.yaml -n argocd
kubectl apply -f argocd/apps/pencraft-prod.yaml -n argocd
```

This tells ArgoCD where to find the Helm chart and which `values-*.yaml` to use.

---

## Step 6 — Verify ArgoCD sees the apps

Open the ArgoCD UI and you should see three apps: `pencraft-dev`, `pencraft-preprod`, `pencraft-prod`.

Click each one and press **Sync** to deploy.

ArgoCD will create Deployments and Services in the target namespaces.

---

## Step 7 — Set up CI (GitHub Actions)

Your application repo must build images and update the `pencraft-gitops` repo. High level actions:

1. Build Docker images and push to Artifact Registry.
2. Update `envs/<env>/values-<env>.yaml` in `pencraft-gitops` with the new image tag or digest.
3. Commit and push to the environment branch (`dev`, `preprod`, or `release`).

We provided working GitHub workflows earlier (dev, preprod, prod). Make sure secrets exist in your repo:

* `GCP_PROJECT_ID`, `GCP_REGION`, `GCP_SA_KEY` (or Workload Identity secrets)
* `GITOPS_PAT` (a GitHub token that can push to `pencraft-gitops`)

---

## Step 8 — Test the pipeline

1. Push a change to your app repo on branch `preprod` (or run workflow manually if `workflow_dispatch` is enabled).
2. CI builds images and updates `envs/preprod/values-preprod.yaml` in `pencraft-gitops`.
3. ArgoCD sees the commit on branch `preprod` and syncs the `pencraft-preprod` Application.
4. Check Kubernetes pods in `pencraft-preprod`:

```bash
kubectl get pods -n pencraft-preprod
kubectl logs -n pencraft-preprod <pod-name>
```

---

## Step 9 — Rollback and troubleshooting

* To rollback, change the `image` in `envs/<env>/values-<env>.yaml` to a previous tag and commit. ArgoCD will redeploy.
* Check ArgoCD UI for sync errors and resource diffs.
* Check controller logs:

```bash
kubectl -n argocd logs deploy/argocd-application-controller
```

---

## Step 10 — Security tips

* Do NOT store secrets in git in plain text.
* Use SealedSecrets or ExternalSecrets to manage secrets.
* Use digest (`@sha256:`) for prod images to ensure immutability.
* Limit `GITOPS_PAT` permissions — only allow access to the `pencraft-gitops` repo.

---

## Useful commands summary

* Create namespaces:

  * `kubectl create namespace pencraft-dev` (and others)
* Apply ArgoCD files:

  * `kubectl apply -f argocd/project.yaml -n argocd`
  * `kubectl apply -f argocd/apps/pencraft-preprod.yaml -n argocd`
* Check pods:

  * `kubectl get pods -n pencraft-preprod`
* View ArgoCD logs:

  * `kubectl -n argocd logs deploy/argocd-repo-server`

---

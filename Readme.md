# 🚀 GitOps on EKS with ArgoCD
### From zero to "`git push` = deploy"

> **Goal of this guide:** install ArgoCD on an EKS cluster, prove it works with the official example, then deploy your own Helm charts in two ways:
> **Demo 1** (click through the UI) and **Demo 2** (everything declared in Git with app-of-apps).

---

## 🧠 The 30-second idea

Think of a restaurant 🍽️

| Restaurant | Kubernetes world |
|---|---|
| 📖 The menu | **Git repo**: what *should* exist |
| 🪑 The tables | **EKS cluster**: what *actually* exists |
| 🧑‍🍳 The waiter who keeps checking | **ArgoCD**: compares menu vs tables, fixes the difference |

If someone sneaks in and moves a chair (`kubectl edit`, `kubectl scale`), the waiter puts it back. **Git is the boss, the cluster obeys.**

### Push vs Pull

| | Old way (CI pushes) | GitOps way (ArgoCD pulls) |
|---|---|---|
| Who deploys | GitHub Actions runs `kubectl apply` | ArgoCD, from *inside* the cluster |
| Cluster credentials | Stored in CI 😬 | Never leave the cluster ✅ |
| Drift detection | None | Constant |
| Rollback | Re-run old pipeline | `git revert` |
| Audit trail | Pipeline logs | Git history |

---

## 🗺️ What we are building

```
   You                GitHub                    EKS cluster
    │                    │                  ┌──────────────────────────┐
    │  git push          │                  │  ┌────────────────────┐  │
    ├───────────────────►│   ◄── pulls ──── │  │      ArgoCD        │  │
    │                    │   every ~3 min   │  └─────────┬──────────┘  │
    │                    │                  │            │ applies     │
    │                    │                  │            ▼             │
    │                    │                  │   namespace: demo        │
    │                    │                  │    ├─ myapp   (pods+svc) │
    │                    │                  │    └─ payments(pods+svc) │
                                            └──────────────────────────┘
```

---

## 🧰 Prerequisites

- [ ] EKS cluster is up and nodes are `Ready`
- [ ] `kubectl`, `aws` CLI, `git`, `helm` installed on your machine
- [ ] A GitHub account (we use SSH, no passwords)
- [ ] Nodes of at least `t3.medium` (ArgoCD runs ~7 pods; tiny instances hit the pod limit and pods stay `Pending`)

---

# 📦 PART 1: Install ArgoCD

### Step 1: Point `kubectl` at your cluster
```bash
aws eks update-kubeconfig --region <your-region> --name <your-cluster-name>
kubectl get nodes
```
💡 **Why:** `kubectl` talks to whichever cluster your kubeconfig points at. Confirm nodes are `Ready` before blaming ArgoCD for anything.

### Step 2: Create a namespace
```bash
kubectl create namespace argocd
```
💡 **Why:** keeps ArgoCD's pods, secrets and RBAC in one box that is easy to inspect or delete.

### Step 3: Install ArgoCD
```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
💡 **Why `--server-side`?** One of the CRDs (ApplicationSet) is too big for a normal `kubectl apply`, which stores a copy of the object in an annotation with a size limit. Without the flag you get `metadata.annotations: Too long`. A classic first error!

💡 **Why `stable`?** Fine for learning. In production, pin an exact version so upgrades are deliberate.

### Step 4: Wait for the pods
```bash
kubectl get pods -n argocd -w
```

| Pod | Its job |
|---|---|
| `argocd-server` | API + web UI |
| `argocd-repo-server` | Clones Git, renders Helm/Kustomize |
| `argocd-application-controller` | 🧠 The brain: compares Git vs cluster, syncs |
| `argocd-redis` | Cache |
| `argocd-dex-server` | SSO (unused for now) |
| `argocd-applicationset-controller` | Generates apps from templates (later) |
| `argocd-notifications-controller` | Slack/email alerts (later) |

All `Running`? Press `Ctrl+C`.

### Step 5: Open the UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Open **https://localhost:8080** and accept the self-signed certificate warning.

💡 **Why port-forward?** It is the cheapest way in: no AWS load balancer, no security group decisions. Proper Ingress comes later.

### Step 6: Get the admin password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```
Username: `admin`. Log in to the UI.

💡 **Why base64?** Kubernetes Secrets are *encoded*, not *encrypted*. Anyone who can read the Secret can read the password.

### Step 7: Install the CLI and log in
```bash
# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd /usr/local/bin/argocd && rm argocd

# Mac
brew install argocd
```
Keep the port-forward running in one terminal, and in another:
```bash
argocd login localhost:8080 --username admin --password '<password>' --insecure
argocd account update-password
kubectl -n argocd delete secret argocd-initial-admin-secret
```
💡 `--insecure` is only because of the self-signed certificate. After you set your own password, the initial secret is no longer needed, so delete it.

---

# 🧪 PART 2: Smoke test with the official example repo

Before using our own repos, prove ArgoCD itself is healthy using the ArgoCD team's public example.
If this works and our own apps later fail, we know the problem is *our* repo, not ArgoCD.

Save as `guestbook-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f guestbook-app.yaml
kubectl get applications -n argocd
kubectl get all -n guestbook
```

✅ **Success looks like:** `Synced` + `Healthy` in the UI, with a tree of Deployment → ReplicaSet → Pod and a Service.

### 📖 Read the Application file like a sentence

> *"ArgoCD, please watch the `guestbook` folder of this GitHub repo and keep it deployed into the `guestbook` namespace of this same cluster, automatically."*

| Field | Meaning |
|---|---|
| `metadata.namespace: argocd` | Where the **Application object** lives (always `argocd`) |
| `project: default` | The guardrail group. `default` has **no restrictions**, perfect for a first test, not for production |
| `repoURL` + `path` | Which repo, which folder inside it |
| `targetRevision: HEAD` | Which branch/tag/commit to follow (`HEAD` = default branch) |
| `destination.server` | `https://kubernetes.default.svc` = the cluster ArgoCD itself runs in |
| `destination.namespace` | Where the **workload** is deployed |
| `automated` | Sync without anyone clicking the button |
| `prune: true` | Deleted from Git → deleted from the cluster |
| `selfHeal: true` | Changed by hand in the cluster → reverted to match Git |
| `CreateNamespace=true` | ArgoCD creates the target namespace |
| `finalizers` | Deleting the Application also deletes what it deployed (otherwise resources are orphaned) |

> 🔑 Two namespaces, two jobs: `metadata.namespace` = where the *instruction* lives. `destination.namespace` = where the *app* runs.

### 🎭 Magic trick: self-heal
```bash
kubectl scale deployment guestbook-ui -n guestbook --replicas=3
kubectl get pods -n guestbook -w
```
Replicas jump to 3, ArgoCD notices the drift, and seconds later they snap back to 1. **Git is the source of truth, not what someone typed.**

### 🧹 Clean up
```bash
kubectl delete -f guestbook-app.yaml
```
The finalizer makes ArgoCD remove the guestbook resources as well.

---

# 🗂️ Our own repo: layout

Create a repo called `gitops-repo`:

```
gitops-repo/
├── charts/
│   ├── myapp/
│   │   ├── Chart.yaml
│   │   ├── values.yaml            ← the file you edit day to day
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   └── payments/                  ← added live in Demo 2
└── argocd/                        ← created in Demo 2
    ├── root.yaml
    └── apps/
        ├── myapp.yaml
        └── payments.yaml
```

<details>
<summary>📄 <b>Click to expand: the Helm chart files</b></summary>

**`charts/myapp/Chart.yaml`**
```yaml
apiVersion: v2
name: myapp
description: Demo app for ArgoCD
type: application
version: 0.1.0
appVersion: "1.0.0"
```

**`charts/myapp/values.yaml`**
```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27.0"          # the line you bump to "deploy"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 80

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

**`charts/myapp/templates/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**`charts/myapp/templates/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
```
</details>

**Test locally before pushing:**
```bash
helm lint charts/myapp
helm template myapp charts/myapp
```
💡 `helm template` prints exactly what ArgoCD will apply. If it renders here, it renders in ArgoCD.

> 🧩 **Why `{{ .Release.Name }}` everywhere?** ArgoCD sets the Helm release name to the Application name. Using it for resource names means two services can share one namespace without colliding. (We rely on this in Demo 2.)

> ⚠️ ArgoCD runs `helm template` and applies the result itself. It does **not** run `helm install`, so `helm list` shows nothing. That is normal.

---

# 🎬 DEMO 1: Deploy a chart by clicking through the UI

**What students learn:** repo credentials, what an Application really is, OutOfSync vs Synced, auto-sync, self-heal.

### 1️⃣ Push the repo to GitHub over SSH

Create an **empty** `gitops-repo` on GitHub (no README, no .gitignore), then:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"     # skip if ~/.ssh/id_ed25519 exists
eval "$(ssh-agent -s)" && ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub                             # paste into GitHub → Settings → SSH and GPG keys
ssh -T git@github.com                                 # expect: "Hi <user>! You've successfully authenticated"
```

```bash
cd gitops-repo
git init
git add .
git commit -m "initial chart"
git branch -M main
git remote add origin git@github.com:jadeja-rajpal/gitops-repo.git
git push -u origin main
```

> ⚠️ SSH URLs use a **colon**: `git@github.com:user/repo.git`. A slash (`git@github.com/user/repo.git`) is not valid and gives "does not appear to be a git repository".
> If `origin` already exists: `git remote set-url origin git@github.com:jadeja-rajpal/gitops-repo.git`

### 2️⃣ Give ArgoCD its own key (a deploy key)

Your laptop's SSH key is *yours*. ArgoCD lives in the cluster and needs its own read-only key.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/argocd_gitops_deploy -C "argocd-deploy-key" -N ""
cat ~/.ssh/argocd_gitops_deploy.pub
```

GitHub → repo → **Settings → Deploy keys → Add deploy key** → paste the public key → leave **Allow write access** unchecked.

💡 **Why a deploy key?** If a personal key leaked from the cluster, an attacker would get all your repos. A read-only deploy key opens exactly one repo, read-only.

### 3️⃣ Connect the repo in the UI

**Settings (⚙️) → Repositories → Connect Repo**

| Field | Value |
|---|---|
| Connection method | `VIA SSH` |
| Name | `gitops-repo` |
| Project | `default` |
| Repository URL | `git@github.com:jadeja-rajpal/gitops-repo.git` |
| SSH private key data | output of `cat ~/.ssh/argocd_gitops_deploy` (include `BEGIN`/`END` lines) |

Click **Connect** → look for ✅ **Successful**.

### 4️⃣ Create the app in the UI

**Applications → + New App**

| Field | Value |
|---|---|
| Application Name | `myapp` |
| Project | `default` |
| Sync Policy | **Manual** ← on purpose! |
| Sync Options | ✔ Auto-Create Namespace |
| Repository URL | pick your repo from the dropdown |
| Revision | `main` |
| Path | `charts/myapp` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `demo` |

Click **Create**, then **Edit as YAML** (top right) 👀 and show that the form and the YAML are the same object. The UI is just a front end for a Kubernetes resource.

### 5️⃣ See the gap, then close it

The app shows **OutOfSync / Missing**: Git has stuff, the cluster doesn't yet.

1. Open the app and look at the resource tree.
2. Click **Sync → Synchronize**.
3. Watch: *Progressing* → **Synced + Healthy** ✅

```bash
kubectl get all -n demo
kubectl port-forward svc/myapp -n demo 8081:80      # open http://localhost:8081
```

| Status | Answers the question |
|---|---|
| **Sync** (Synced / OutOfSync) | Does the cluster match Git? |
| **Health** (Healthy / Progressing / Degraded) | Are the pods actually working? |

An app can be **Synced but Degraded**: manifests applied fine, but the image tag doesn't exist.

### 6️⃣ Turn on auto-sync and deploy by committing

**App Details → Sync Policy → Enable Auto-Sync** (tick *Prune* and *Self Heal*).

Edit `charts/myapp/values.yaml`, set `tag: "1.27.1"` (or `replicaCount: 3`), then:
```bash
git add . && git commit -m "bump tag" && git push
```
ArgoCD polls every ~3 minutes. Click **Refresh** to skip the wait, then:
```bash
kubectl get pods -n demo -w      # watch the rolling update
```

### 7️⃣ Self-heal and rollback
```bash
kubectl scale deploy myapp -n demo --replicas=5     # ArgoCD reverts it
```
Rollback = `git revert HEAD && git push`. Compare with `kubectl rollout undo`, which ArgoCD would simply override.

### ✅ Demo 1 takeaways
- The **Application** says *where to look*. The **chart** says *what to run*.
- Deploying = changing `values.yaml` in Git. Nobody touches the cluster.
- The app we just made lives **only in the cluster**. Lose the cluster and the Application is gone. Demo 2 fixes that. 👇

---

# 🎬 DEMO 2: App-of-Apps (the Applications live in Git too)

**The problem with Demo 1:** every new service means clicking in the UI or running `kubectl apply`, which is manual, untracked and unreviewed.

**The fix:** one **root** Application, applied once, that watches a folder full of Application files. Add a service = commit a file.

```
 kubectl apply (once)
        │
        ▼
   ┌─────────┐   watches argocd/apps/   ┌────────────┐
   │  root   │ ───────────────────────► │  myapp.yaml│──► myapp  ──┐
   └─────────┘                          │payments.yaml│──► payments ┼─► namespace: demo
                                        └────────────┘             ─┘
```

### 1️⃣ Remove the UI-made app (avoid a name clash)
```bash
argocd app delete myapp
```
Keep the `demo` namespace, we reuse it.

### 2️⃣ Create the files

**`argocd/root.yaml`**: sits *outside* `argocd/apps/`, otherwise root would try to manage itself.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:jadeja-rajpal/gitops-repo.git
    targetRevision: main
    path: argocd/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd          # child Application objects must live in argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**`argocd/apps/myapp.yaml`**: plain, hardcoded, no variables.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:jadeja-rajpal/gitops-repo.git
    targetRevision: main
    path: charts/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 3️⃣ Push first, then apply root
```bash
git add .
git commit -m "add root app and myapp"
git push

kubectl apply -f argocd/root.yaml        # ← the ONLY manual apply, ever
```
> ⚠️ Push **before** applying. Root reads from GitHub, not from your laptop. Unpushed files = root finds nothing.

### 4️⃣ Verify
```bash
kubectl get applications -n argocd       # root + myapp, both Synced/Healthy
kubectl get deploy,svc,pods -n demo
```
In the UI, `root` shows `myapp` as its child.

### 5️⃣ ✨ The magic moment: add a second service with zero `kubectl`

```bash
cp -r charts/myapp charts/payments
sed -i 's/^name: myapp/name: payments/' charts/payments/Chart.yaml
cp argocd/apps/myapp.yaml argocd/apps/payments.yaml
```

Edit `argocd/apps/payments.yaml`, changing exactly **two** places:
```yaml
metadata:
  name: payments               # ① 
...
    path: charts/payments      # ② 
```
(`destination.namespace` stays `demo`: both services share one namespace.)

```bash
git add . && git commit -m "add payments" && git push
```
Click **Refresh** on `root`. A `payments` app appears in the UI. You never ran `kubectl apply` for it. 🤯

```bash
kubectl get deploy,svc -n demo           # myapp AND payments side by side
```

> **Naming rule for students:** folder name = Application name = resource names. The namespace is shared. Because templates use `{{ .Release.Name }}`, `myapp` and `payments` never collide.

### 6️⃣ Independent lifecycles
- Change `replicaCount` in `charts/payments/values.yaml` and push. Only `payments` rolls; `myapp` is untouched.
- `kubectl scale deploy myapp -n demo --replicas=5` gets reverted; `payments` is unaffected.

### 7️⃣ Reverse demo: deleting
Delete `argocd/apps/payments.yaml` and push. The `payments` **Application disappears**, but its pods **keep running**, because the file has no finalizer.
To make deletion also remove the workload, add under `metadata:`
```yaml
  finalizers:
    - resources-finalizer.argocd.argoproj.io
```
Many production teams leave it off on purpose, so deleting a file by mistake doesn't take down a live service. Teach both.

### 🔁 Do I re-apply root?

| Change | Re-apply root? |
|---|---|
| Edit or add an app file | ❌ just push |
| Delete an app file | ❌ just push |
| Edit `root.yaml` itself | ✅ root doesn't manage itself |

---

## 🧯 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Pods `Pending` in `argocd` | Node too small / pod limit | Use `t3.medium` or bigger |
| `metadata.annotations: Too long` | Missed `--server-side` | Re-apply with `--server-side --force-conflicts` |
| Repo shows **Failed** | Deploy key not added on GitHub, or key pasted without `BEGIN`/`END` | Add `.pub` under Deploy keys, re-paste private key |
| `does not appear to be a git repository` | `/` instead of `:` in SSH URL | `git@github.com:user/repo.git` |
| `ComparisonError` | Wrong `path` or Helm template error | Run `helm template charts/<name>` locally |
| Stays **OutOfSync** after push | Haven't refreshed yet | Click **Refresh** (polling is ~3 min) |
| `ImagePullBackOff` | Bad tag, or nodes can't reach the registry | Check the tag; check NAT gateway / egress |
| Root finds nothing | Files not pushed, or wrong `path` | `git push`; path must be `argocd/apps` |
| Two apps fight over one name | UI app and Git app both called `myapp` | `argocd app delete myapp`, then Refresh `root` |

---

## 📋 Command cheat sheet

```bash
# Status
kubectl get applications -n argocd
argocd app list
argocd app get <name>

# Actions
argocd app sync <name>
argocd app delete <name>
argocd app history <name>

# Debug
kubectl get pods -n argocd
kubectl logs -n argocd deploy/argocd-repo-server
kubectl describe application <name> -n argocd
```

---

## 🏆 Golden rules

1. 📖 **Git is the source of truth.** The cluster follows.
2. 🚫 Never fix things with `kubectl` by hand. Self-heal will undo it anyway.
3. 🔖 A deploy is a **change to `values.yaml`**, never to the Application file.
4. 🧾 One commit = one deployment. Git history is your deploy history.
5. 🥾 `root.yaml` is applied **once**. Everything after that is a `git push`.
6. 🔐 Never put plain Secrets in Git (use External Secrets Operator or Sealed Secrets).
7. 🏷️ Never deploy with the `latest` tag. Use a unique tag per build (e.g. the commit SHA).

---

## 🔭 What's next

- [ ] `AppProject` restrictions: limit which repos and namespaces an app may use
- [ ] ApplicationSet: one template that generates an Application per folder
- [ ] Separate app repo + GitHub Actions building the image to ECR (OIDC, no stored keys)
- [ ] Environments: `values-dev.yaml` / `values-prod.yaml`
- [ ] Ingress + SSO for the ArgoCD UI
- [ ] GitHub webhook for instant sync instead of 3-minute polling
- [ ] Notifications to Slack, sync waves, Argo Rollouts

---

⭐ *If you can explain the restaurant analogy to a friend, you understand GitOps.*

# ocp-platform

GitOps-Plattform-Repo für OpenShift Local (CRC).  
Verwaltet vom **Platform Team**.

---

## Übersicht

```
ocp-platform/               ← dieses Repo (Platform Team)
  bootstrap/                ← einmalig manuell ausführen
  apps/                     ← Child-Apps, von ArgoCD verwaltet
  cluster-config/           ← cluster-weite Konfiguration, von ArgoCD verwaltet

ocp-workloads/              ← Workloads-Repo (Platform Team)
  charts/namespace-config/  ← Helm Chart für Namespace-Konfiguration
  apps/<project>/           ← je Projekt: AppProject, Gruppen, App-Referenzen

<app-repo>/                 ← je App ein eigenes Repo (Entwickler-Team)
  helm/
```

---

## Sync-Flow

```
platform-app  (Root App-of-Apps, Bootstrap)
├── cluster-config  ──────────→ cluster-config/               Wave -1
│   ├── argocd/
│   │   └── argocd-rbac-cm.yaml      ArgoCD RBAC (von ArgoCD verwaltet)
│   ├── groups/
│   │   └── cluster-admins.yaml
│   ├── oauth/
│   │   └── oauth.yaml               OAuth (von ArgoCD verwaltet)
│   └── rbac/
│       └── argocd-cluster-admin.yaml
│
└── workloads-app  ───────────→ ocp-workloads/apps/           Wave 0
    └── project-a/
        ├── appproject.yaml                                   Wave -1
        ├── groups.yaml                                       Wave -1
        ├── my-app/
        │   ├── namespace-config-app.yaml                     Wave  0
        │   └── my-app-app.yaml                               Wave  1
        └── your-app/
            ├── namespace-config-app.yaml                     Wave  0
            └── your-app-app.yaml                             Wave  1
```

---

## Repo-Struktur

```
ocp-platform/
├── bootstrap/
│   ├── gitops-operator/
│   │   ├── namespace.yaml
│   │   ├── operatorgroup.yaml
│   │   └── subscription.yaml
│   ├── argocd-rbac-cm.yaml    ← nur Bootstrap (danach via cluster-config verwaltet)
│   ├── oauth.yaml             ← nur Bootstrap (danach via cluster-config verwaltet)
│   ├── platform-project.yaml  AppProject "platform"
│   ├── platform-app.yaml      Root App-of-Apps
│   └── README-bootstrap.md    Schritt-für-Schritt Bootstrap-Anleitung
├── apps/
│   ├── cluster-config-app.yaml    Wave -1 → cluster-config/
│   └── workloads-app.yaml         Wave  0 → ocp-workloads/apps/
└── cluster-config/
    ├── argocd/
    │   └── argocd-rbac-cm.yaml    ArgoCD RBAC: cluster-admins → ArgoCD admin
    ├── groups/
    │   └── cluster-admins.yaml    Gruppe cluster-admins
    ├── oauth/
    │   └── oauth.yaml             HTPasswd Identity Provider
    └── rbac/
        └── argocd-cluster-admin.yaml  ClusterRoleBinding für ArgoCD
```

---

## User- und Gruppen-Management

| Was | Wo | Wer |
|---|---|---|
| Cluster-weite Gruppen (z.B. `cluster-admins`) | `cluster-config/groups/` | Platform Team |
| Projekt-Gruppen (z.B. `project-a-admins`) | `ocp-workloads/apps/<project>/groups.yaml` | Platform Team |
| Passwörter / HTPasswd-Hashes | **nicht in Git** – manueller Schritt | Platform Team |
| User-Objekte | entstehen automatisch beim ersten Login | — |

### Neuen Platform-Admin hinzufügen

**1. User in Git zur Gruppe hinzufügen** (`cluster-config/groups/cluster-admins.yaml`):

```yaml
users:
  - admin
  - neuer-admin
```

```powershell
git add . && git commit -m "feat(groups): add neuer-admin to cluster-admins"
git push
```

**2. Passwort manuell im HTPasswd-Secret ergänzen:**

```powershell
oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) | `
  Out-File -FilePath "$env:TEMP\htpasswd" -Encoding utf8NoBOM

# Hash generieren: https://bcrypt-generator.com (Rounds 10)
Add-Content "$env:TEMP\htpasswd" 'neuer-admin:$2a$10$HASH_HIER'

oc create secret generic htpasswd-secret `
  --from-file=htpasswd="$env:TEMP\htpasswd" `
  -n openshift-config `
  --dry-run=client -o yaml | oc apply -f -

Remove-Item "$env:TEMP\htpasswd"
```

---

## Weiterführendes

- [bootstrap/README-bootstrap.md](bootstrap/README-bootstrap.md) — Bootstrap-Anleitung
- [ocp-workloads](https://github.com/chriwo42-lang/ocp-workloads) — Workloads-Repo

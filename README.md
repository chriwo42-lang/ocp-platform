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
  groups/                   ← Globale Gruppen-Definitionen (eine Datei pro Gruppe)
  charts/namespace-config/  ← Helm Chart für Namespace-Konfiguration
  apps/<project>/           ← je Projekt: AppProject + App-Referenzen

<app-repo>/                 ← je App ein eigenes Repo (Entwickler-Team)
  helm/
```

---

## Sync-Flow

```
platform-app  (Root App-of-Apps, Bootstrap)
├── cluster-config  ──────────────→ cluster-config/          Wave -1
│   ├── argocd/argocd-rbac-cm.yaml
│   ├── groups/cluster-admins.yaml
│   ├── oauth/oauth.yaml
│   └── rbac/argocd-cluster-admin.yaml
│
├── workloads-groups-app  ─────────→ ocp-workloads/groups/   Wave -1
│   ├── project-a-admins.yaml
│   └── project-a-developers.yaml
│
└── workloads-app  ────────────────→ ocp-workloads/apps/     Wave 0
    └── project-a/
        ├── appproject.yaml                                  Wave -1
        ├── my-app/
        │   ├── namespace-config-app.yaml                    Wave  0
        │   └── my-app-app.yaml                              Wave  1
        └── your-app/
            ├── namespace-config-app.yaml                    Wave  0
            └── your-app-app.yaml                            Wave  1
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
│   └── README-bootstrap.md
├── apps/
│   ├── cluster-config-app.yaml       Wave -1 → cluster-config/
│   ├── workloads-groups-app.yaml     Wave -1 → ocp-workloads/groups/
│   └── workloads-app.yaml            Wave  0 → ocp-workloads/apps/
└── cluster-config/
    ├── argocd/
    │   └── argocd-rbac-cm.yaml
    ├── groups/
    │   └── cluster-admins.yaml
    ├── oauth/
    │   └── oauth.yaml
    └── rbac/
        └── argocd-cluster-admin.yaml
```

---

## Gruppen-Konzept

| Gruppe | Definiert in | Zweck |
|---|---|---|
| `cluster-admins` | `ocp-platform/cluster-config/groups/` | Platform-weite Admins |
| `project-a-admins` | `ocp-workloads/groups/` | Admins für project-a Namespaces |
| `project-a-developers` | `ocp-workloads/groups/` | Entwickler für project-a Namespaces |

Gruppen sind **globale Ressourcen** — sie werden einmal definiert und in den jeweiligen  
`values.yaml` der Apps unter `rbac.adminGroups` / `rbac.editGroups` referenziert.

---

## Neuen Platform-Admin hinzufügen

**1.** `cluster-config/groups/cluster-admins.yaml` editieren:

```yaml
users:
  - admin
  - neuer-admin
```

```powershell
git add . && git commit -m "feat(groups): add neuer-admin"
git push
```

**2.** Passwort im HTPasswd-Secret ergänzen:

```powershell
oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) | `
  Out-File -FilePath "$env:TEMP\htpasswd" -Encoding utf8NoBOM

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

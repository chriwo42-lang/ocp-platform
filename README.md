# ocp-platform

GitOps-Plattform-Repo für OpenShift Local (CRC).  
Verwaltet vom **Platform Team**.

---

## Übersicht

```
ocp-platform/               ← dieses Repo (Platform Team)
  bootstrap/                ← einmalig manuell ausführen
  apps/                     ← Child-Apps + platform-app selbst, von ArgoCD verwaltet
  cluster-config/           ← cluster-weite Konfiguration, von ArgoCD verwaltet

ocp-workloads/              ← Workloads-Repo (Platform Team)
  apps/
    groups/                 ← Team-Definitionen (eine Datei pro Team)
    <project>/              ← je Projekt: AppProject + App-Referenzen
  charts/namespace-config/  ← Helm Chart für Namespace-Konfiguration

<app-repo>/                 ← je App ein eigenes Repo (Entwickler-Team)
  helm/
```

---

## ArgoCD RBAC

Der OpenShift GitOps Operator setzt folgende Policy automatisch:

```
g, system:cluster-admins, role:admin
g, cluster-admins, role:admin
policy.default: role:readonly
```

Die Gruppe `cluster-admins` hat damit ohne weitere Konfiguration ArgoCD admin-Rechte.  
Es ist **keine** manuelle RBAC-Konfiguration notwendig.

---

## Sync-Flow

```
platform-app  (Root App-of-Apps — verwaltet sich selbst via apps/platform-app.yaml)
├── platform-app.yaml  ────────────→ sich selbst                   (kein Wave)
├── cluster-config  ──────────────→ cluster-config/                Wave -1
│   ├── oauth/oauth.yaml
│   └── rbac/
│       ├── groups/cluster-admins.yaml
│       ├── argocd-cluster-admin.yaml
│       └── cluster-admins-cluster-admin.yaml
│
└── workloads-app  ────────────────→ ocp-workloads/apps/           Wave  0
    ├── groups/
    │   ├── team-a.yaml                                            Wave -1
    │   └── team-b.yaml                                            Wave -1
    ├── project-a/
    │   ├── appproject.yaml                                        Wave -1
    │   ├── my-app/
    │   │   ├── namespace-config-app.yaml                          Wave  0
    │   │   └── my-app-app.yaml                                    Wave  1
    │   └── your-app/
    │       ├── namespace-config-app.yaml                          Wave  0
    │       └── your-app-app.yaml                                  Wave  1
    └── project-b/
        ├── appproject.yaml                                        Wave -1
        ├── my-app/
        │   ├── namespace-config-app.yaml                          Wave  0
        │   └── my-app-app.yaml                                    Wave  1
        └── your-app/
            ├── namespace-config-app.yaml                          Wave  0
            └── your-app-app.yaml                                  Wave  1
```

> **Sync Waves** steuern die Deploy-Reihenfolge **innerhalb einer Application**.
> `cluster-config` (Wave -1) und `workloads-app` (Wave 0) sind Waves innerhalb von `platform-app`.
> Die internen Waves in `workloads-app` sind davon unabhängig.
> Details: [bootstrap/README-bootstrap.md](bootstrap/README-bootstrap.md#schritt-7--argocd-sync-abwarten)

---

## Repo-Struktur

```
ocp-platform/
├── bootstrap/
│   ├── gitops-operator/
│   │   ├── namespace.yaml
│   │   ├── operatorgroup.yaml
│   │   └── subscription.yaml
│   ├── platform-project.yaml      AppProject "platform" (nur Bootstrap, nicht in Git verwaltet)
│   └── README-bootstrap.md
│
│   HINWEIS: platform-app.yaml liegt nicht mehr im bootstrap/ Ordner.
│   Bootstrap-Befehl: oc apply -f apps/platform-app.yaml
│
├── apps/                          Von ArgoCD verwaltet (recurse: true)
│   ├── platform-app.yaml          Root App-of-Apps (self-managed)
│   ├── cluster-config-app.yaml    Wave -1 → cluster-config/
│   └── workloads-app.yaml         Wave  0 → ocp-workloads/apps/
└── cluster-config/
    ├── oauth/
    │   └── oauth.yaml
    └── rbac/
        ├── groups/
        │   └── cluster-admins.yaml
        ├── argocd-cluster-admin.yaml
        └── cluster-admins-cluster-admin.yaml
```

---

## Gruppen-Konzept

| Gruppe | Definiert in | Zweck |
|---|---|---|
| `cluster-admins` | `ocp-platform/cluster-config/rbac/groups/` | Platform-weite Admins + ArgoCD admin |
| `team-a` | `ocp-workloads/apps/groups/` | Entwickler-Team, Zuweisung je App in `values.yaml` |
| `team-b` | `ocp-workloads/apps/groups/` | Entwickler-Team, Zuweisung je App in `values.yaml` |

Teams sind **projektunabhängig** — ein Team kann in verschiedenen Projekten unterschiedliche  
Rollen haben (admin, edit, view). Die Zuweisung erfolgt in `values.yaml` je App.

**ArgoCD AppProject-Rollen** werden ausschließlich von `cluster-admins` verwaltet —  
Entwickler-Teams haben keinen direkten ArgoCD-Zugriff.

---

## Neuen Platform-Admin hinzufügen

**1.** `cluster-config/rbac/groups/cluster-admins.yaml` editieren:

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

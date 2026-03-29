# ocp-platform

GitOps-Plattform-Repo für OpenShift Local (CRC).  
Verwaltet vom **Platform Team**.

---

## Übersicht

```
ocp-platform/               ← dieses Repo (Platform Team)
  bootstrap/                ← einmalig manuell ausführen
  apps/                     ← Child-Apps, von ArgoCD verwaltet
  cluster-config/           ← cluster-weite Konfiguration (Gruppen, ...)

ocp-workloads/              ← Workloads-Repo (Platform Team)
  charts/namespace-config/  ← Helm Library Chart für Namespace-Konfiguration
  apps/<project>/           ← je Projekt: AppProject, Gruppen, App-Referenzen

<app-repo>/                 ← je App ein eigenes Repo (Entwickler-Team)
  helm/ oder manifests/
```

---

## Sync-Flow

```
platform-app  (Root App-of-Apps, Bootstrap)
├── cluster-config  ──────────→ cluster-config/          Wave 0
│   └── groups/
│       └── cluster-admins.yaml
│
└── workloads-app  ───────────→ ocp-workloads/apps/      Wave 0
    └── project-a/
        ├── appproject.yaml                              Wave -1
        ├── groups.yaml                                  Wave  0
        └── my-app/
            ├── namespace-config-app.yaml               Wave  0
            └── my-app-app.yaml                         Wave  1
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
│   ├── argocd-rbac-cm.yaml        ArgoCD RBAC: cluster-admins → ArgoCD admin
│   ├── platform-project.yaml      AppProject "platform"
│   ├── platform-app.yaml          Root App-of-Apps
│   └── README-bootstrap.md        Schritt-für-Schritt Bootstrap-Anleitung
├── apps/
│   ├── cluster-config-app.yaml    Child-App → cluster-config/
│   └── workloads-app.yaml         Child-App → ocp-workloads/apps/
└── cluster-config/
    └── groups/
        └── cluster-admins.yaml    Gruppe cluster-admins (platform-weite Admins)
```

---

## User- und Gruppen-Management

### Konzept

| Was | Wo | Wer |
|---|---|---|
| Cluster-weite Gruppen (z.B. `cluster-admins`) | `cluster-config/groups/` | Platform Team |
| Projekt-Gruppen (z.B. `project-a-admins`) | `ocp-workloads/apps/<project>/groups.yaml` | Platform Team |
| Passwörter / HTPasswd-Hashes | **nicht in Git** – manueller Bootstrap-Schritt | Platform Team |
| User-Objekte | entstehen automatisch beim ersten Login | — |

### Neuen Platform-Admin hinzufügen

```bash
# cluster-config/groups/cluster-admins.yaml editieren:
users:
  - admin
  - neuer-admin    # hinzufügen

git add . && git commit -m "feat(groups): add neuer-admin to cluster-admins" && git push
```

Passwort für den neuen User manuell im HTPasswd-Secret ergänzen:

```bash
# Bestehenden Hash auslesen
oc get secret htpasswd-secret -n openshift-config \
  -o jsonpath='{.data.htpasswd}' | base64 -d > /tmp/htpasswd

# Neuen User hinzufügen (htpasswd CLI oder bcrypt-generator.com)
htpasswd /tmp/htpasswd neuer-admin

# Secret aktualisieren
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=/tmp/htpasswd \
  -n openshift-config \
  --dry-run=client -o yaml | oc apply -f -

rm /tmp/htpasswd
```

---

## Weiterführendes

- [bootstrap/README-bootstrap.md](bootstrap/README-bootstrap.md) — Bootstrap-Anleitung
- [ocp-workloads](https://github.com/chriwo42-lang/ocp-workloads) — Workloads-Repo

# Bootstrap – Initiales Setup

Alle Schritte in diesem Dokument werden **einmalig manuell** ausgeführt.  
Danach übernimmt ArgoCD die vollständige Verwaltung über Git.

---

## Voraussetzungen

```powershell
crc status          # Cluster muss Running sein
oc whoami           # muss "kubeadmin" sein
```

---

## Schritt 1 – GitOps Operator installieren

```powershell
oc apply -f bootstrap\gitops-operator\
oc rollout status deployment/openshift-gitops-server -n openshift-gitops --timeout=300s
```

---

## Schritt 2 – HTPasswd Secret anlegen

BCrypt-Hashes generieren (Rounds 10): https://bcrypt-generator.com

Alle User in einem einzigen Secret anlegen:

```powershell
oc create secret generic htpasswd-secret `
  --from-literal=htpasswd='admin:$2a$12$n2KLKFwn/Yvu6kHOMKwKa.OrEXChWJi8.uHz2q6SEV3lJspaxd98W
developer:$2a$12$hlleQYCSvrfu98AfUQ1La.vW8sv2vc/ml1oxbcyNIamvxLNL4kjwG
editor:$2a$12$pWRcY/JsRPmT0qWqdpdyMenPhCKdU7paeLzwrj5WD8HdnH7q6jSNC
readonly:$2a$12$LqYGf7O7ZOqSWbYI7dOnE.znJzlwOhOHyxHHDCbpwB78fOsngmiLu' `
  -n openshift-config
```

> Das Secret enthält Passwörter – es wird **nicht in Git gespeichert**.

---

## Schritt 3 – OAuth auf HTPasswd umstellen

```powershell
oc apply -f cluster-config\oauth\oauth.yaml
oc rollout status deployment/oauth-openshift -n openshift-authentication --timeout=120s
```

> Die OAuth-Konfiguration liegt in `cluster-config/oauth/oauth.yaml` und wird
> nach dem ersten ArgoCD-Sync von dort verwaltet.

---

## Schritt 4 – Admin-User einrichten

Alle Ressourcen werden direkt aus Git angewendet.  
ArgoCD übernimmt sie beim ersten Sync ohne Konflikt.

```powershell
# Gruppe mit admin-User anlegen
oc apply -f cluster-config\rbac\groups\cluster-admins.yaml

# cluster-admin Berechtigung für die Gruppe
oc apply -f cluster-config\rbac\cluster-admins-cluster-admin.yaml

# cluster-admin Berechtigung für den ArgoCD Application Controller
oc apply -f cluster-config\rbac\argocd-cluster-admin.yaml
```

Login testen:

```powershell
oc login -u admin -p <dein-passwort> https://api.crc.testing:6443
oc whoami   # muss "admin" zurückgeben
```

> **ArgoCD RBAC:** Der OpenShift GitOps Operator setzt `g, cluster-admins, role:admin`
> automatisch. Kein manueller Schritt notwendig.

---

## Schritt 5 – AppProject "platform" anlegen

```powershell
oc apply -f bootstrap\platform-project.yaml
```

> `platform-project.yaml` bleibt dauerhaft in `bootstrap/` — das AppProject kann nicht
> von der Application verwaltet werden die es selbst nutzt.

---

## Schritt 6 – Root App-of-Apps anlegen

```powershell
oc apply -f apps\platform-app.yaml
```

Ab hier übernimmt ArgoCD. Alle weiteren Änderungen erfolgen **ausschließlich über Git**.

> `platform-app.yaml` liegt in `apps/` — nicht in `bootstrap/`. Sie wird einmalig manuell
> angewendet und danach von ArgoCD selbst verwaltet (self-managed).
> Änderungen an `platform-app` bitte in `apps/platform-app.yaml` vornehmen.

---

## Schritt 7 – ArgoCD Sync abwarten

ArgoCD deployt `platform-app` und dessen Child-Apps automatisch.

> **Wie Sync Waves funktionieren:** Waves steuern die Deploy-Reihenfolge
> **innerhalb einer einzigen ArgoCD Application**. Jede Application hat
> ihren eigenen Wave-Kontext — die Waves in `cluster-config` und
> `workloads-app` sind voneinander unabhängig.

**Ebene 1 — innerhalb `platform-app`** (steuert Child-App Reihenfolge):

| Wave | Child-App | Warum diese Reihenfolge |
|---|---|---|
| — | `platform-app` | Verwaltet sich selbst, kein Wave nötig |
| -1 | `cluster-config` | `argocd-cluster-admin` CRB muss existieren bevor `workloads-app` Ressourcen in anderen Namespaces anlegt |
| 0 | `workloads-app` | Startet erst wenn `cluster-config` vollständig synced ist |

**Ebene 2 — innerhalb `cluster-config`** (unabhängig von Ebene 1):

| Wave | Ressource | Warum diese Reihenfolge |
|---|---|---|
| -1 | `cluster-admins` Gruppe | Muss vor dem ClusterRoleBinding existieren das sie referenziert |
| 0 | ClusterRoleBindings, OAuth | Referenzieren die Gruppe aus Wave -1 |

**Ebene 2 — innerhalb `workloads-app`** (unabhängig von Ebene 1):

| Wave | Ressource | Warum diese Reihenfolge |
|---|---|---|
| -1 | Teams (Gruppen) | Müssen vor RoleBindings existieren die sie referenzieren |
| -1 | AppProjects | Müssen vor Applications existieren die sie referenzieren |
| 0 | App-Config | Namespace, Quota, NetPol, RBAC + Application für App-Repo |

ArgoCD UI aufrufen:

```powershell
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

Login mit `admin` / `<dein-passwort>`.

Folgende Apps müssen `Synced / Healthy` sein:

| App | Beschreibung |
|---|---|
| `platform-app` | Root App-of-Apps |
| `cluster-config` | OAuth, Gruppen, RBAC |
| `workloads-app` | Alle Workload-Apps |
| `project-a-appproject` | AppProject project-a |
| `project-a-my-app-config` | Namespace + App (project-a/my-app) |
| `project-a-your-app-config` | Namespace + App (project-a/your-app) |
| `project-b-appproject` | AppProject project-b |
| `project-b-my-app-config` | Namespace + App (project-b/my-app) |
| `project-b-your-app-config` | Namespace + App (project-b/your-app) |
| `project-a-my-app` | my-app Workload in project-a |
| `project-a-your-app` | your-app Workload in project-a |
| `project-b-my-app` | my-app Workload in project-b |
| `project-b-your-app` | your-app Workload in project-b |

---

## Neuen User anlegen

**1. Gruppe in Git pflegen** (`ocp-workloads/apps/groups/<team>.yaml`):

```yaml
users:
  - neuer-user
```

```powershell
git add . && git commit -m "feat(groups): add neuer-user to team-x"
git push
```

**2. Passwort direkt im Secret ergänzen** (kein Tempfile):

```powershell
# Bestehenden Inhalt lesen
$existing = oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))

# Hash generieren: https://bcrypt-generator.com (Rounds 10)
# Zeile anhängen: neuer-user:$2a$10$HASH
$combined = "$existing`nneuer-user:`$2a`$10`$HASH_HIER"

$encoded = [System.Convert]::ToBase64String(
  [System.Text.Encoding]::UTF8.GetBytes($combined)
)

oc patch secret htpasswd-secret -n openshift-config `
  --type merge `
  -p "{`"data`":{`"htpasswd`":`"$encoded`"}}"
```

---

## Troubleshooting

**Apps zeigen "forbidden" Fehler:**

```powershell
oc apply -f cluster-config\rbac\argocd-cluster-admin.yaml
```

**OAuth funktioniert nicht — Inhalt prüfen:**

```powershell
oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))
```

**`cluster-config` App zeigt OutOfSync für Group oder ClusterRoleBinding:**  
Normal beim ersten Sync — ArgoCD übernimmt die in Schritt 4 angelegten Ressourcen.

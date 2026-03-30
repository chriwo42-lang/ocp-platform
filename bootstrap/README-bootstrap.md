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

BCrypt-Hash generieren (Rounds 10): https://bcrypt-generator.com

```powershell
oc create secret generic htpasswd-secret `
  --from-literal=htpasswd='admin:$2a$10$HASH_HIER_EINSETZEN' `
  -n openshift-config
```

> Das Secret enthält ein Passwort – es wird **nicht in Git gespeichert**.

---

## Schritt 3 – OAuth auf HTPasswd umstellen

```powershell
oc apply -f bootstrap\oauth.yaml
oc rollout status deployment/oauth-openshift -n openshift-authentication --timeout=120s
```

> Danach wird `oauth.yaml` von ArgoCD via `cluster-config/oauth/oauth.yaml` verwaltet.

---

## Schritt 4 – Admin-User temporär einrichten

Gruppe und ClusterRoleBindings werden nur temporär manuell angelegt.  
Danach verwaltet ArgoCD beides aus Git (`cluster-config/groups/` und `cluster-config/rbac/`).

```powershell
# Gruppe anlegen und admin hinzufügen
oc adm groups new cluster-admins
oc adm groups add-users cluster-admins admin

# ClusterRoleBindings anlegen (werden von ArgoCD später aus Git übernommen)
oc apply -f cluster-config\rbac\cluster-admins-cluster-admin.yaml
oc apply -f cluster-config\rbac\argocd-cluster-admin.yaml
```

Login testen:

```powershell
oc login -u admin -p <dein-passwort> https://api.crc.testing:6443
oc whoami   # muss "admin" zurückgeben
```

> **ArgoCD RBAC:** Der OpenShift GitOps Operator setzt `g, cluster-admins, role:admin`
> automatisch in der `argocd-rbac-cm`. Kein manueller Schritt notwendig.

---

## Schritt 5 – AppProject "platform" anlegen

```powershell
oc apply -f bootstrap\platform-project.yaml
```

---

## Schritt 6 – Root App-of-Apps anlegen

```powershell
oc apply -f bootstrap\platform-app.yaml
```

Ab hier übernimmt ArgoCD. Alle weiteren Änderungen erfolgen **ausschließlich über Git**.

---

## Schritt 7 – ArgoCD Sync abwarten

ArgoCD deployt nun automatisch alle Child-Apps in der richtigen Reihenfolge:

| Wave | App / Ressource | Was wird deployt |
|---|---|---|
| -1 | `cluster-config` | OAuth, `cluster-admins` Gruppe, ClusterRoleBindings |
| -1 | `workloads-groups-app` | Gruppen für alle Projekte |
| -1 | AppProjects | AppProject je Projekt |
| 0 | `workloads-app` | Namespace-Config Apps |
| 1 | App-of-Apps | Eigentliche Workloads |

ArgoCD UI aufrufen:

```powershell
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

Login mit `admin` / `<dein-passwort>`.

Folgende Apps müssen `Synced / Healthy` sein:

| App | Quelle |
|---|---|
| `platform-app` | ocp-platform/apps/ |
| `cluster-config` | ocp-platform/cluster-config/ |
| `workloads-groups-app` | ocp-workloads/groups/ |
| `workloads-app` | ocp-workloads/apps/ |
| `project-a-my-app-namespace` | ocp-workloads charts/namespace-config |
| `project-a-my-app` | my-app/helm |
| `project-a-your-app-namespace` | ocp-workloads charts/namespace-config |
| `project-a-your-app` | your-app/helm |
| `project-b-my-app-namespace` | ocp-workloads charts/namespace-config |
| `project-b-my-app` | my-app/helm |
| `project-b-your-app-namespace` | ocp-workloads charts/namespace-config |
| `project-b-your-app` | your-app/helm |

---

## Neuen User anlegen

**1. Gruppe in Git pflegen** (`ocp-workloads/groups/<project>/<rolle>.yaml`):

```yaml
users:
  - neuer-user
```

```powershell
git add . && git commit -m "feat(groups): add neuer-user"
git push
```

**2. Passwort manuell im Secret ergänzen:**

```powershell
oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) | `
  Out-File -FilePath "$env:TEMP\htpasswd" -Encoding utf8NoBOM

# Hash generieren: https://bcrypt-generator.com (Rounds 10)
Add-Content "$env:TEMP\htpasswd" 'neuer-user:$2a$10$HASH_HIER'

oc create secret generic htpasswd-secret `
  --from-file=htpasswd="$env:TEMP\htpasswd" `
  -n openshift-config `
  --dry-run=client -o yaml | oc apply -f -

Remove-Item "$env:TEMP\htpasswd"
```

---

## Troubleshooting

**Apps zeigen "forbidden" Fehler:**

```powershell
oc apply -f cluster-config\rbac\argocd-cluster-admin.yaml
```

**OAuth funktioniert nicht — Hash prüfen:**

```powershell
oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))
```

**`cluster-config` App zeigt OutOfSync für Group oder ClusterRoleBinding:**  
Normal beim ersten Sync — ArgoCD übernimmt die temporär angelegten Ressourcen aus Schritt 4.

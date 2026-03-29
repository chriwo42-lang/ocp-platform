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

---

## Schritt 4 – Admin-User temporär einrichten

Dieser Schritt ist nur bis zum ersten ArgoCD-Sync nötig.  
Danach verwaltet ArgoCD die Gruppe `cluster-admins` aus Git.

```powershell
oc adm groups new cluster-admins
oc adm groups add-users cluster-admins admin
oc adm policy add-cluster-role-to-user cluster-admin admin
```

Login testen:

```powershell
oc login -u admin -p <dein-passwort> https://api.crc.testing:6443
oc whoami   # muss "admin" zurückgeben
```

---

## Schritt 5 – ArgoCD RBAC setzen

```powershell
oc apply -f bootstrap\argocd-rbac-cm.yaml
```

Mappt die Gruppe `cluster-admins` auf die ArgoCD admin-Rolle.

---

## Schritt 6 – AppProject "platform" anlegen

```powershell
oc apply -f bootstrap\platform-project.yaml
```

---

## Schritt 7 – Root App-of-Apps anlegen

```powershell
oc apply -f bootstrap\platform-app.yaml
```

Ab hier übernimmt ArgoCD. Alle weiteren Änderungen erfolgen **ausschließlich über Git**.

---

## Schritt 8 – ArgoCD Sync abwarten

ArgoCD deployt nun automatisch alle Child-Apps in der richtigen Reihenfolge:

| Wave | App | Was wird deployt |
|---|---|---|
| 0 | `cluster-config` | Gruppe `cluster-admins` aus Git |
| 0 | `workloads-app` | AppProjects, Gruppen, Namespace-Config, Apps |

ArgoCD URL aufrufen:

```powershell
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

Login mit `admin` / `<dein-passwort>` in der ArgoCD UI.

Folgende Apps müssen `Synced / Healthy` sein:

| App | Quelle |
|---|---|
| `platform-app` | ocp-platform/apps/ |
| `cluster-config` | ocp-platform/cluster-config/ |
| `workloads-app` | ocp-workloads/apps/ |

---

## Neuen User anlegen

**1. Gruppe in Git pflegen:**

```yaml
# ocp-workloads/apps/project-a/groups.yaml
users:
  - neuer-user
```

**2. Passwort manuell im Secret ergänzen (PowerShell):**

```powershell
# Bestehenden Hash auslesen
oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) | `
  Out-File -FilePath "$env:TEMP\htpasswd" -Encoding utf8NoBOM

# Neuen Eintrag manuell anhängen (Hash von https://bcrypt-generator.com):
# Format: username:$2a$10$HASH
Add-Content "$env:TEMP\htpasswd" "neuer-user:`$2a`$10`$HASH_HIER"

# Secret aktualisieren
oc create secret generic htpasswd-secret `
  --from-file=htpasswd="$env:TEMP\htpasswd" `
  -n openshift-config `
  --dry-run=client -o yaml | oc apply -f -

Remove-Item "$env:TEMP\htpasswd"
```

---

## Troubleshooting

**ArgoCD zeigt "Unknown" für admin:**

```powershell
oc apply -f bootstrap\argocd-rbac-cm.yaml
oc rollout restart deployment/openshift-gitops-server -n openshift-gitops
```

**OAuth funktioniert nicht — Hash prüfen:**

```powershell
oc get secret htpasswd-secret -n openshift-config `
  -o jsonpath='{.data.htpasswd}' | `
  [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))
```

**`cluster-config` App zeigt OutOfSync für Group:**  
Normal beim ersten Sync — ArgoCD übernimmt die temporär angelegte Gruppe aus Schritt 4.

# Bootstrap – Initiales Setup

Alle Schritte in diesem Dokument werden **einmalig manuell** ausgeführt.  
Danach übernimmt ArgoCD die vollständige Verwaltung über Git.

---

## Voraussetzungen

```bash
crc status          # Cluster muss Running sein
oc whoami           # muss "kubeadmin" sein
```

---

## Schritt 1 – GitOps Operator installieren

```bash
oc apply -f bootstrap/gitops-operator/
```

Warten bis ArgoCD vollständig gestartet ist:

```bash
oc rollout status deployment/openshift-gitops-server -n openshift-gitops --timeout=180s
```

---

## Schritt 2 – HTPasswd Secret anlegen

BCrypt-Hash generieren (Rounds 10, z.B. unter https://bcrypt-generator.com):

```bash
oc create secret generic htpasswd-secret \
  --from-literal=htpasswd='admin:$2a$10$HASH_HIER_EINSETZEN' \
  -n openshift-config
```

> Das Secret enthält ein Passwort – es wird **nicht in Git gespeichert**.

---

## Schritt 3 – OAuth auf HTPasswd umstellen

```bash
oc patch oauth cluster --type merge -p '{
  "spec": {
    "identityProviders": [{
      "name": "htpasswd",
      "mappingMethod": "claim",
      "type": "HTPasswd",
      "htpasswd": {
        "fileData": {
          "name": "htpasswd-secret"
        }
      }
    }]
  }
}'
```

Warten bis der OAuth-Server neu startet:

```bash
oc rollout status deployment/oauth-openshift -n openshift-authentication --timeout=120s
```

---

## Schritt 4 – Admin-User temporär einrichten

Dieser Schritt ist nur bis zum ersten ArgoCD-Sync nötig.  
Danach verwaltet ArgoCD die Gruppe `cluster-admins` aus Git.

```bash
# Temporäre Gruppe anlegen (wird von ArgoCD in Schritt 8 übernommen)
oc adm groups new cluster-admins
oc adm groups add-users cluster-admins admin

# cluster-admin Berechtigung für admin
oc adm policy add-cluster-role-to-user cluster-admin admin
```

Login testen:

```bash
oc login -u admin -p <dein-passwort> https://api.crc.testing:6443
oc whoami   # muss "admin" zurückgeben
```

---

## Schritt 5 – ArgoCD RBAC setzen

```bash
oc apply -f bootstrap/argocd-rbac-cm.yaml
```

Mappt die Gruppe `cluster-admins` auf die ArgoCD admin-Rolle.

---

## Schritt 6 – AppProject "platform" anlegen

```bash
oc apply -f bootstrap/platform-project.yaml
```

---

## Schritt 7 – Root App-of-Apps anlegen

```bash
oc apply -f bootstrap/platform-app.yaml
```

Ab hier übernimmt ArgoCD. Alle weiteren Änderungen erfolgen **ausschließlich über Git**.

---

## Schritt 8 – ArgoCD Sync abwarten

ArgoCD deployt nun automatisch alle Child-Apps in der richtigen Reihenfolge:

| Wave | App | Was wird deployt |
|---|---|---|
| 0 | `cluster-config` | Gruppe `cluster-admins` aus Git (übernimmt temporäre Gruppe aus Schritt 4) |
| 0 | `workloads-app` | Lädt `ocp-workloads/apps/` — deployt AppProjects, Gruppen, Apps |

Sync-Status prüfen:

```bash
# ArgoCD URL ausgeben
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

Nach dem Bootstrap werden User **nicht** mehr manuell per `oc adm` angelegt.  
Der Ablauf ist zweigeteilt:

**1. Gruppe in Git pflegen** (z.B. `ocp-workloads/apps/project-a/groups.yaml`):

```yaml
users:
  - neuer-user
```

**2. Passwort manuell im Secret ergänzen:**

```bash
oc get secret htpasswd-secret -n openshift-config \
  -o jsonpath='{.data.htpasswd}' | base64 -d > /tmp/htpasswd

htpasswd /tmp/htpasswd neuer-user

oc create secret generic htpasswd-secret \
  --from-file=htpasswd=/tmp/htpasswd \
  -n openshift-config \
  --dry-run=client -o yaml | oc apply -f -

rm /tmp/htpasswd
```

---

## Troubleshooting

**ArgoCD zeigt "Unknown" für admin:**

```bash
oc apply -f bootstrap/argocd-rbac-cm.yaml
oc rollout restart deployment/openshift-gitops-server -n openshift-gitops
```

**OAuth funktioniert nicht:**

```bash
oc get secret htpasswd-secret -n openshift-config \
  -o jsonpath='{.data.htpasswd}' | base64 -d
```

**`cluster-config` App zeigt OutOfSync für Group:**  
ArgoCD versucht die Gruppe zu überschreiben. Das ist normal beim ersten Sync —  
ArgoCD übernimmt die temporär angelegte Gruppe aus Schritt 4 und verwaltet sie ab jetzt aus Git.

# OCP Platform

Dieses Repository enthält die deklarative Konfiguration für die OpenShift Container Platform. Alle Komponenten werden als YAML-Manifeste verwaltet und per `oc apply` oder über GitOps auf den Cluster angewendet.

## Voraussetzungen

- Zugang zu einem OpenShift-Cluster (z. B. CRC für lokale Entwicklung)
- `oc` CLI installiert und eingeloggt als Benutzer mit `cluster-admin`-Rolle
- Der Cluster hat Zugriff auf den Red Hat Operator Catalog (`redhat-operators`)

## Komponenten

### OpenShift GitOps Operator

Der [Red Hat OpenShift GitOps Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.10/html/installing_gitops/installing-openshift-gitops) stellt eine vollständig verwaltete ArgoCD-Instanz bereit. Er bildet die Grundlage für das GitOps-basierte Management aller weiteren Plattform- und Workload-Konfigurationen.

Die Installation erfolgt über drei Manifeste:

| Manifest | Beschreibung |
|---|---|
| `namespace.yaml` | Erstellt den Namespace `openshift-gitops-operator` für den Operator |
| `operatorgroup.yaml` | Definiert die OperatorGroup, die den Installationsscope festlegt |
| `subscription.yaml` | Erstellt die Subscription, die den Operator über OLM aus dem `redhat-operators`-Catalog installiert |

#### Installation

Die Manifeste werden in der richtigen Reihenfolge angewendet:

```bash
oc apply -f components/openshift-gitops-operator/namespace.yaml
oc apply -f components/openshift-gitops-operator/operatorgroup.yaml
oc apply -f components/openshift-gitops-operator/subscription.yaml
```

#### Prüfen, ob die Installation erfolgreich war

Die Operator-Pods im Operator-Namespace prüfen:

```bash
oc -n openshift-gitops-operator get po
```

Die ArgoCD-Pods im GitOps-Namespace prüfen — der Operator erstellt hier automatisch eine einsatzbereite ArgoCD-Instanz:

```bash
oc -n openshift-gitops get po
```

Sobald alle Pods im Status `Running` sind, ist die ArgoCD-Oberfläche erreichbar unter:

```
https://openshift-gitops-server-openshift-gitops.apps-crc.testing
```

> **Hinweis:** Das initiale Admin-Passwort für ArgoCD kann mit folgendem Befehl ausgelesen werden:
>
> ```bash
> oc -n openshift-gitops extract secret/openshift-gitops-cluster --to=-
> ```

## Verzeichnisstruktur

```
ocp-platform/
├── components/
│   └── openshift-gitops-operator/
│       ├── namespace.yaml
│       ├── operatorgroup.yaml
│       └── subscription.yaml
└── README.md
```

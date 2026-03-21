# 1. GitOps Operator installieren

# 2. GitHub Repo-Secret für ArgoCD
oc create secret generic ocp-gitops-bootstrap-repo \
  --namespace=openshift-gitops \
  --from-literal=type=git \
  --from-literal=url=https://github.com/<user>/ocp-gitops-bootstrap.git \
  --from-literal=username=<github-user> \
  --from-literal=password=<PAT>

oc label secret ocp-gitops-bootstrap-repo \
  -n openshift-gitops \
  argocd.argoproj.io/secret-type=repository

# 3. HTPasswd Secret
oc create secret generic htpasswd-secret \
  --from-literal=htpasswd='admin:<bcrypt-hash>' \
  -n openshift-config

# 4. Root-App anwenden
oc apply -f bootstrap/root-app.yaml

# 5. Admin zur cluster-admins Gruppe hinzufügen
oc adm groups new cluster-admins
oc adm groups add-users cluster-admins admin
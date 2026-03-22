# installPlanApproval
in git bei den operator subscriptions auf automatic setzen

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

# 4. Root-App 
oc apply -f .\project-platform.yaml
oc apply -f bootstrap/root-app.yaml

# 5. Admin zur cluster-admins Gruppe hinzufügen
oc adm groups new cluster-admins
oc adm groups add-users cluster-admins admin

# vault init
oc exec -it vault-0 -n vault -- vault operator init
keys speichern!!!
oc exec -it vault-0 -n vault -- vault operator unseal <unseal-key-1>
oc exec -it vault-0 -n vault -- vault operator unseal <unseal-key-2>
oc exec -it vault-0 -n vault -- vault operator unseal <unseal-key-3>

# eso mit vault verbinden
oc exec -it vault-0 -n vault -- vault login <root-token>

oc exec -it vault-0 -n vault -- vault policy write eso-policy - 
path "secret/*" {
  capabilities = ["read", "list"]
}

oc exec -it vault-0 -n vault -- vault token create -policy=eso-policy -period=768h
oc create secret generic vault-token -n openshift-external-secrets-operator --from-literal=token=<vault-token>

# 6. installPlanApproval
in git bei den operator subscriptions auf manual setzen

# kubeadmin entfernen???
oc delete secret kubeadmin -n kube-system
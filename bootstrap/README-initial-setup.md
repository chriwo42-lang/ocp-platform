# 0. installPlanApproval
in git bei den operator subscriptions auf automatic setzen

# autosync enablen
in den apps in git

# 1. GitOps Operator installieren
oc apply -f ... <- files aus dem git operators ordner

# 2. GitHub Repo-Secret für ArgoCD
oc create secret generic ocp-platform-repo --namespace=openshift-gitops --from-literal=type=git   --from-literal=url=https://github.com/chriwo42-lang/ocp-platform.git --from-literal=username=chriwo42-lang  --from-literal=password=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

oc label secret ocp-platform-repo -n openshift-gitops argocd.argoproj.io/secret-type=repository

# 3. HTPasswd Secret
oc create secret generic htpasswd-secret --from-literal=htpasswd='admin:$2a$12$u5tzoqjBFxHmviyVzkMOp.zXpagO9rD19APwEkMgbZB/3dX/OLTRG' -n openshift-config

# 4. Root-App 
oc apply -f .\project-platform.yaml
oc apply -f bootstrap/root-app.yaml

# 5. Admin zur cluster-admins Gruppe hinzufügen
oc adm groups new cluster-admins
oc adm groups add-users cluster-admins admin


# 6. installPlanApproval
in git bei den operator subscriptions auf manual setzen

# autosync disablen
in den apps in git

# kubeadmin entfernen???
oc delete secret kubeadmin -n kube-system
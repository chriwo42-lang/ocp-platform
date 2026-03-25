# installPlanApproval
in git bei den operator subscriptions auf automatic setzen

# autosync enablen
in den apps in git

# GitOps Operator installieren
oc apply -f ... <- files aus dem ordner operators/openshift-gitops-operator

# GitHub Repo-Secret für ArgoCD
oc create secret generic ocp-platform-repo --namespace=openshift-gitops --from-literal=type=git   --from-literal=url=https://github.com/chriwo42-lang/ocp-platform.git --from-literal=username=chriwo42-lang  --from-literal=password=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

oc label secret ocp-platform-repo -n openshift-gitops argocd.argoproj.io/secret-type=repository

# HTPasswd Secret admin user (kubeadmin sieht die projekte in argo nicht per default)
oc create secret generic htpasswd-secret --from-literal=htpasswd='admin:$2a$12$u5tzoqjBFxHmviyVzkMOp.zXpagO9rD19APwEkMgbZB/3dX/OLTRG' -n openshift-config
oc adm groups new cluster-admins (TODO: wirklich nötig? prüfen ob gruppe nicht automatisch angelegt wird)
oc adm groups add-users cluster-admins admin

# Root-App 
oc apply -f .\project-platform.yaml
oc apply -f bootstrap/root-app.yaml


# vault initialisieren
oc exec -it vault-0 -n vault -- vault operator init
oc exec -it vault-0 -n vault -- vault operator unseal <key1>
oc exec -it vault-0 -n vault -- vault operator unseal <key2>
oc exec -it vault-0 -n vault -- vault operator unseal <key3>

oc exec -it vault-0 -n vault -- vault login <root-token>
oc exec -it vault-0 -n vault -- vault secrets enable -path=secret kv-v2

oc exec -it vault-0 -n vault -- /bin/sh
cat > /tmp/eso-policy.hcl << 'EOF'
> path "secret/*" {
>   capabilities = ["read", "list"]
> }
> EOF

vault policy write eso-policy /tmp/eso-policy.hcl

oc exec -it vault-0 -n vault -- vault token create -policy=eso-policy -period=768h

# installPlanApproval
in git bei den operator subscriptions auf manual setzen

# autosync disablen
in den apps in git

# kubeadmin entfernen???
oc delete secret kubeadmin -n kube-system

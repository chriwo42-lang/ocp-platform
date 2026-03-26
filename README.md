# ocp-platform

README in directory bootstrap


# known issues

## vault unseal
oc exec -it vault-0 -n vault -- vault operator unseal key1
oc exec -it vault-0 -n vault -- vault operator unseal key2
oc exec -it vault-0 -n vault -- vault operator unseal key3

oc annotate clustersecretstore vault-backend force-sync=$(Get-Date -UFormat %s) --overwrite

## operator syncen nicht
oc annotate subscription openshift-cert-manager-operator -n cert-manager-operator olm.operatorframework.io/force-update=$(Get-Date -UFormat %s) --overwrite

oc annotate subscription compliance-operator -n openshift-compliance olm.operatorframework.io/force-update=$(Get-Date -UFormat %s) --overwrite
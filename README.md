# ocp-platform

README in directory bootstrap


# known issues

## vault unseal

## operator syncen nicht
oc annotate subscription openshift-cert-manager-operator -n cert-manager-operator olm.operatorframework.io/force-update=$(Get-Date -UFormat %s) --overwrite

oc annotate subscription compliance-operator -n openshift-compliance olm.operatorframework.io/force-update=$(Get-Date -UFormat %s) --overwrite
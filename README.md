---
title: Sample Operator & Air-gapped
linktitle: Sample Operator
description: How to sync, all or parial sample operator images
tags:
  - air-gapped
  - disconnected
  - restricted-network
---

# Sample Operator in a air-gapped enviorment

First of all it is importand to configure your local image registry propperly
in case your local registry use a private signed certificate:

```
oc create configmap registry-config --from-file=${LOCAL_REGISTRY/:/..}=/var/lib/libvirt/images/mirror-registry/certs/ca.crt -n openshift-config

oc patch image.config.openshift.io/cluster --patch '{"spec":{"additionalTrustedCA":{"name":"registry-config"}}}' --type=merge
```

To get a full list of examples/images you wanna sync enable the samples operator:

```
oc patch configs.samples.operator.openshift.io/cluster \
  --type='json' \
  --patch='[{"op": "replace", "path": "/spec/managementState", "value": "Managed"},{"op": "replace", "path": "/spec/samplesRegistry", "value": "host.compute.local:5000/samples"}]'
```

Get the list of images:

=== "Command"

    ```
    oc get is -n openshift -o json | jq -r '.items[].spec.tags[].from.name' | grep registry.redhat.io
    ```


=== "Example list of an 4.9.11 cluster"

    [Download the list image-list-4.9.11.txt]({{ page.canonical_url }}../image-list-4.9.11.txt)

    ```
    --8<-- "content/cluster-installation/air-gapped/image-list-4.9.11.txt"
    ```


Sync selected images:

```bash
oc get is -n openshift -o json \
  | jq -r '.items[].spec.tags[].from.name' \
  | grep registry.redhat.io \
  > image-list-4.9.11.txt

cat image-list-4.9.11.txt | grep -E '(openjdk-11|openjdk-18|java)'  | sort -u | while read line;
do
  echo $line   ${LOCAL_REGISTRY}/samples/${line//registry.redhat.io\/} | tee -a image-to-sync
done

```

Part of `images-to-sync`
```txt
registry.redhat.io/redhat-openjdk-18/openjdk18-openshift:1.5 host.compute.local:5000/samples/redhat-openjdk-18/openjdk18-openshift:1.5
registry.redhat.io/redhat-openjdk-18/openjdk18-openshift:1.6 host.compute.local:5000/samples/redhat-openjdk-18/openjdk18-openshift:1.6
registry.redhat.io/redhat-openjdk-18/openjdk18-openshift:1.7 host.compute.local:5000/samples/redhat-openjdk-18/openjdk18-openshift:1.7
registry.redhat.io/redhat-openjdk-18/openjdk18-openshift:1.8 host.compute.local:5000/samples/redhat-openjdk-18/openjdk18-openshift:1.8
registry.redhat.io/redhat-openjdk-18/openjdk18-openshift:latest host.compute.local:5000/samples/redhat-openjdk-18/openjdk18-openshift:latest
registry.redhat.io/ubi8/openjdk-11:latest host.compute.local:5000/samples/ubi8/openjdk-11:latest
```

Sync images:

```
oc image mirror -a $LOCAL_SECRET_JSON -f images-to-sync
```


Adjust sample operator image:


ImageStreams:
```
LIST=$(oc get is -n openshift -o go-template='{{range .items}}"{{.metadata.name}}"{{"\n"}}{{end}}'|grep -v -E '(openjdk-11|openjdk-18|java)' | tr "\n" ',' | sed 's/,$//')

oc patch configs.samples.operator.openshift.io/cluster \
  -n openshift-cluster-samples-operator \
  --patch "{\"spec\":{\"skippedImagestreams\":[$LIST] } }"  \
  --type=merge

oc delete is -n openshift -l samples.operator.openshift.io/managed --wait=false
```

Templates:
```
LIST=$(oc get templates -n openshift -o go-template='{{range .items}}"{{.metadata.name}}"{{"\n"}}{{end}}' | tr "\n" ',' | sed 's/,$//')

oc patch configs.samples.operator.openshift.io/cluster \
  -n openshift-cluster-samples-operator \
  --patch "{\"spec\":{\"skippedTemplates\":[$LIST] } }"  \
  --type=merge

oc delete templates -n openshift -l samples.operator.openshift.io/managed --wait=false
```

Check the operator logs:
```
oc logs -f -n openshift-cluster-samples-operator -l name=cluster-samples-operator -c cluster-samples-operator
```


Official documentation:

<https://docs.openshift.com/container-platform/4.9/openshift_images/samples-operator-alt-registry.html#installation-restricted-network-samples_samples-operator-alt-registry>


### Helmchart repo

oc delete helmchartrepositories/openshift-helm-charts
helmchartrepository.helm.openshift.io "openshift-helm-charts" deleted


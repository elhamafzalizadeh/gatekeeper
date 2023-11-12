# Policies

## container-must-have-limits-and-requests

This Policy requires containers to have defined resources set (limits and requests for Memory & CPU). https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

## container-must-have-probes

This Policy requires Pods to have readiness and liveness probes.

## workload-must-have-label

This Policy requires workloads to contain specified labels, with values matching provided regular expressions.
    
## namespace-must-have-label.yaml

This Policy requires namespace to contain specified labels.

## container-image-must-not-have-latest-tag

Requires container images to have an image tag different from the ones in the specified list.(latest tag) https://kubernetes.io/docs/concepts/containers/images/#image-names
# Gatekeeper

The Open Policy Agent (OPA) is an open-source, general-purpose policy engine that unifies policy enforcement across the stack.

OPA provides a high-level declarative language that lets us specify policies as code and simple APIs to offload policy decision-making from our software.

We can use OPA to enforce policies in Kubernetes.OPA Gatekeeper is a specialized project providing first-class integration between OPA and Kubernetes.


# Steps:

* 1-Installing Gatekeeper
* 2-ConstraintTemplates
* 3-policies


## 1-Installing Gatekeeper

For installing gatekeeper follow these steps:

### Clone the repository:

```
git clone https://github.com/elhamafzalizadeh/gatekeeper.git
cd installation-manifest
sudo kubectl apply -f gatekeeper.yaml

```
Now check the running pods of gatekeeper-system namespace!

## 2-ConstraintTemplates

For each policy(constraint) you should define the ConstraintTemplates first (rego source).For example if you want create a policy to require resources to contain specified labels,
You should create the ConstraintTemplate which names k8srequiredlabels.

```
cd ../ConstraintTemplate
sudo kubectl apply -f ConstraintTemplate-k8srequiredlabels.yaml

```

<details>
<summary><b>View:ConstraintTemplate-k8srequiredlabels.yaml</b></summary>
<br>

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
  annotations:
    metadata.gatekeeper.sh/title: "Required Labels"
    metadata.gatekeeper.sh/version: 1.0.0
    description: >-
      Requires resources to contain specified labels, with values matching
      provided regular expressions.
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            message:
              type: string
            labels:
              type: array
              description: >-
                A list of labels and values the object must specify.
              items:
                type: object
                properties:
                  key:
                    type: string
                    description: >-
                      The required label.
                  allowedRegex:
                    type: string
                    description: >-
                      If specified, a regular expression the annotation's value
                      must match. The value must contain at least one match for
                      the regular expression.
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        get_message(parameters, _default) = msg {
          not parameters.message
          msg := _default
        }

        get_message(parameters, _default) = msg {
          msg := parameters.message
        }

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_].key}
          missing := required - provided
          count(missing) > 0
          def_msg := sprintf("you must provide labels: %v", [missing])
          msg := get_message(input.parameters, def_msg)
        }

        violation[{"msg": msg}] {
          value := input.review.object.metadata.labels[key]
          expected := input.parameters.labels[_]
          expected.key == key
          # do not match if allowedRegex is not defined, or is an empty string
          expected.allowedRegex != ""
          not re_match(expected.allowedRegex, value)
          def_msg := sprintf("Label <%v: %v> does not satisfy allowed regex: %v", [key, value, expected.allowedRegex])
          msg := get_message(input.parameters, def_msg)
        }
```


<br>
</details>

---

## 3-policies

### Policy for labels

Now you can create a policy for k8srequiredlabels.Create policy for namespace first and then workloads.

#### Policy for namespace label

```
cd ../policies
sudo kubectl apply -f ns-must-have-label.yaml

```


<details>
<summary><b>View:ns-must-have-label.yaml</b></summary>
<br>

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
  parameters:
    labels:
    - app.kubernetes.io/environment
    - app.kubernetes.io/appname


```

<br>
</details>

---

#### Policy for workloads label

```
sudo kubectl apply -f workload-must-have-label.yaml

```

<details>
<summary><b>View:workload-must-have-label.yaml</b></summary>
<br>

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: workload-must-have-label
spec:
  match:
    kinds:
      - apiGroups: ["*"]
        kinds:
        - Pod
        - Deployment
        - ReplicaSet
        - StatefulSet
        - DaemonSet
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
  parameters:
    labels:
    - app.kubernetes.io/environment
    - app.kubernetes.io/apptype
    - app.kubernetes.io/appname
    - app.kubernetes.io/version

```

<br>
</details>

---

For each policy,create the ConstraintTemplates first and then create the policy(constraint) of that template.


### Policy for RequiredProbes

Another example is K8sRequiredProbes policy.This policy Requires Pods to have readiness and/or liveness probes.

```
cd ../ConstraintTemplate
sudo kubectl apply -f ConstraintTemplate-k8srequiredprobes.yaml

```

<details>
<summary><b>View:ConstraintTemplate-k8srequiredprobes.yaml</b></summary>
<br>

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredprobes
  annotations:
    metadata.gatekeeper.sh/title: "Required Probes"
    metadata.gatekeeper.sh/version: 1.0.1
    description: Requires Pods to have readiness and/or liveness probes.
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredProbes
      validation:
        openAPIV3Schema:
          type: object
          properties:
            probes:
              description: "A list of probes that are required (ex: `readinessProbe`)"
              type: array
              items:
                type: string
            probeTypes:
              description: "The probe must define a field listed in `probeType` in order to satisfy the constraint (ex. `tcpSocket` satisfies `['tcpSocket', 'exec']`)"
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredprobes

        import data.lib.exclude_update.is_update

        probe_type_set = probe_types {
            probe_types := {type | type := input.parameters.probeTypes[_]}
        }

        violation[{"msg": msg}] {
            # Probe fields are immutable.
            not is_update(input.review)

            container := input.review.object.spec.containers[_]
            probe := input.parameters.probes[_]
            probe_is_missing(container, probe)
            msg := get_violation_message(container, input.review, probe)
        }

        probe_is_missing(ctr, probe) = true {
            not ctr[probe]
        }

        probe_is_missing(ctr, probe) = true {
            probe_field_empty(ctr, probe)
        }

        probe_field_empty(ctr, probe) = true {
            probe_fields := {field | ctr[probe][field]}
            diff_fields := probe_type_set - probe_fields
            count(diff_fields) == count(probe_type_set)
        }

        get_violation_message(container, review, probe) = msg {
            msg := sprintf("Container <%v> in your <%v> <%v> has no <%v>", [container.name, review.kind.kind, review.object.metadata.name, probe])
        }
      libs:
        - |
          package lib.exclude_update

          is_update(review) {
              review.operation == "UPDATE"
          }

```

<br>
</details>

---

```
cd policies
sudo kubectl apply -f container-must-have-probes.yaml 

```


<details>
<summary><b>View:container-must-have-probes.yaml</b></summary>
<br>

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredProbes
metadata:
  name: container-must-have-probes
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod","Deployment","ReplicaSet","StatefulSet","DaemonSet"]
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
  parameters:
    probes: ["readinessProbe", "livenessProbe"]
    probeTypes: ["tcpSocket", "httpGet", "exec"]
```

<br>
</details>

---

### Policy for RequiredResources(Container limit&request)

```
cd ../ConstraintTemplate
sudo kubectl apply -f ConstraintTemplate-k8srequiredresources.yaml

```

<details>
<summary><b>View:ConstraintTemplate-k8srequiredresources.yaml</b></summary>
<br>

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
  annotations:
    metadata.gatekeeper.sh/title: "Required Resources"
    metadata.gatekeeper.sh/version: 1.0.1
    description: >-
      Requires containers to have defined resources set.

      https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          properties:
            exemptImages:
              description: >-
                Any container that uses an image that matches an entry in this list will be excluded
                from enforcement. Prefix-matching can be signified with `*`. For example: `my-image-*`.

                It is recommended that users use the fully-qualified Docker image name (e.g. start with a domain name)
                in order to avoid unexpectedly exempting images from an untrusted repository.
              type: array
              items:
                type: string
            limits:
              type: array
              description: "A list of limits that should be enforced (`cpu`, `memory`, or both)."
              items:
                type: string
                enum:
                - cpu
                - memory
            requests:
              type: array
              description: "A list of requests that should be enforced (`cpu`, `memory`, or both)."
              items:
                type: string
                enum:
                - cpu
                - memory
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        import data.lib.exempt_container.is_exempt

        violation[{"msg": msg}] {
          general_violation[{"msg": msg, "field": "containers"}]
        }

        violation[{"msg": msg}] {
          general_violation[{"msg": msg, "field": "initContainers"}]
        }

        general_violation[{"msg": msg, "field": field}] {
          container := input.review.object.spec[field][_]
          not is_exempt(container)
          provided := {resource_type | container.resources.limits[resource_type]}
          required := {resource_type | resource_type := input.parameters.limits[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("container <%v> does not have <%v> limits defined", [container.name, missing])
        }

        general_violation[{"msg": msg, "field": field}] {
          container := input.review.object.spec[field][_]
          not is_exempt(container)
          provided := {resource_type | container.resources.requests[resource_type]}
          required := {resource_type | resource_type := input.parameters.requests[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("container <%v> does not have <%v> requests defined", [container.name, missing])
        }
      libs:
        - |
          package lib.exempt_container

          is_exempt(container) {
              exempt_images := object.get(object.get(input, "parameters", {}), "exemptImages", [])
              img := container.image
              exemption := exempt_images[_]
              _matches_exemption(img, exemption)
          }

          _matches_exemption(img, exemption) {
              not endswith(exemption, "*")
              exemption == img
          }

          _matches_exemption(img, exemption) {
              endswith(exemption, "*")
              prefix := trim_suffix(exemption, "*")
              startswith(img, prefix)
          }

```

<br>
</details>

```
cd ../policies
sudo kubectl apply -f container-must-have-limits-and-requests.yaml

```

<details>
<summary><b>View:/container-must-have-limits-and-requests.yaml</b></summary>
<br>

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: container-must-have-limits-and-requests
spec:
  match:
    kinds:
      - apiGroups: ["*"]
        kinds:
        - Pod
        - Deployment
        - ReplicaSet
        - StatefulSet
        - DaemonSet
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
  parameters:
    limits:
      - cpu
      - memory
    requests:
      - cpu
      - memory

```

<br>
</details>

### Policy for Disallow-latest-tag

```
cd ../ConstraintTemplate
sudo kubectl apply -f ConstraintTemplate-k8sdisallowedtags.yaml

```

<details>
<summary><b>View:ConstraintTemplate-k8sdisallowedtags.yaml</b></summary>
<br>

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowedtags
  annotations:
    metadata.gatekeeper.sh/title: "Disallow tags"
    metadata.gatekeeper.sh/version: 1.0.0
    description: >-
      Requires container images to have an image tag different from the ones in
      the specified list.

      https://kubernetes.io/docs/concepts/containers/images/#image-names
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowedTags
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          properties:
            exemptImages:
              description: >-
                Any container that uses an image that matches an entry in this list will be excluded
                from enforcement. Prefix-matching can be signified with `*`. For example: `my-image-*`.
                It is recommended that users use the fully-qualified Docker image name (e.g. start with a domain name)
                in order to avoid unexpectedly exempting images from an untrusted repository.
              type: array
              items:
                type: string
            tags:
              type: array
              description: Disallowed container image tags.
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowedtags

        import data.lib.exempt_container.is_exempt

        violation[{"msg": msg}] {
            container := input_containers[_]
            not is_exempt(container)
            tags := [forbid | tag = input.parameters.tags[_] ; forbid = endswith(container.image, concat(":", ["", tag]))]
            any(tags)
            msg := sprintf("container <%v> uses a disallowed tag <%v>; disallowed tags are %v", [container.name, container.image, input.parameters.tags])
        }

        violation[{"msg": msg}] {
            container := input_containers[_]
            not is_exempt(container)
            tag := [contains(container.image, ":")]
            not all(tag)
            msg := sprintf("container <%v> didn't specify an image tag <%v>", [container.name, container.image])
        }

        input_containers[c] {
            c := input.review.object.spec.containers[_]
        }
        input_containers[c] {
            c := input.review.object.spec.initContainers[_]
        }
        input_containers[c] {
            c := input.review.object.spec.ephemeralContainers[_]
        }
      libs:
        - |
          package lib.exempt_container

          is_exempt(container) {
              exempt_images := object.get(object.get(input, "parameters", {}), "exemptImages", [])
              img := container.image
              exemption := exempt_images[_]
              _matches_exemption(img, exemption)
          }

          _matches_exemption(img, exemption) {
              not endswith(exemption, "*")
              exemption == img
          }

          _matches_exemption(img, exemption) {
              endswith(exemption, "*")
              prefix := trim_suffix(exemption, "*")
              startswith(img, prefix)
          }

```

<br>
</details>

```
cd ../policies
sudo kubectl apply -f container-image-must-not-have-latest-tag.yaml

```

<details>
<summary><b>View:container-image-must-not-have-latest-tag.yaml</b></summary>
<br>

```yaml

apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowedTags
metadata:
  name: container-image-must-not-have-latest-tag
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod","Deployment","ReplicaSet","StatefulSet","DaemonSet"]
  parameters:
    tags: ["latest"]


<br>
</details>

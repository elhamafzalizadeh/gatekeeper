# Gatekeeper

The Open Policy Agent (OPA) is an open-source, general-purpose policy engine that unifies policy enforcement across the stack.

OPA provides a high-level declarative language that lets us specify policies as code and simple APIs to offload policy decision-making from our software.

We can use OPA to enforce policies in Kubernetes.We have created the label policies according to the platform document.

OPA Gatekeeper is a specialized project providing first-class integration between OPA and Kubernetes.

# Steps:

* 1-Installing Gatekeeper
* 2-ConstraintTemplates
* 3-policies


## 1-Installing Gatekeeper

For installing gatekeeper follow these steps:

### Clone the repository:

```
git clone https://github.com/elhamafzalizadeh/gatekeeper.git
cd gatekeeper/production/installation-manifest
sudo kubectl apply -f gatekeeper.yaml

```

## 2-ConstraintTemplates

For each policy(constraint) you should define the ConstraintTemplates first (rego source).for example if you want create a policy to requires resources to contain specified labels,
You should create the ConstraintTemplate which names k8srequiredlabels.

```
cd ConstraintTemplate
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

Now you can create a policy for k8srequiredlabels.create policy for namespace first and then apply the policy for the workloads.

```
cd policies
sudo kubectl apply -f namespace-must-have-label.yaml

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

sudo kubectl apply -f workload-must-have-label.yaml

```

```

sudo kubectl apply -f namespace-must-have-label.yaml

```
<details>
<summary><b>View:ns-must-have-label.yaml</b></summary>
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
        - Ingress
        - CronJob
        - Service
        - Secret
        - Job
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
Now check the running pod of gatekeeper-system namespace:

# Gatekeeper

The Open Policy Agent (OPA) is an open-source, general-purpose policy engine that unifies policy enforcement across the stack.

OPA provides a high-level declarative language that lets us specify policies as code and simple APIs to offload policy decision-making from our software.

We can use OPA to enforce policies in Kubernetes.We have created the label policies according to the platform document.

OPA Gatekeeper is a specialized project providing first-class integration between OPA and Kubernetes.

# Steps:

* 1-Installing Gatekeeper
* 2-ConstraintTemplates
* 3-policies


## Installing Gatekeeper

For installing gatekeeper follow these steps:

### Clone the repository:

```
git clone https://github.com/elhamafzalizadeh/gatekeeper.git
cd gatekeeper/production/installation-manifest
sudo kubectl apply -f gatekeeper.yaml

```

## ConstraintTemplates





Now check the running pod of gatekeeper-system namespace:

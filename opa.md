### OPA - Open Policy Agent

An external tool that allows to write custom policies for the cluster. The OPA Gatekeeper will make the OPA to act as an admission controller and the basic configuration would be to first create a `ConstraintTemplate` and then a `Constraint`, which will leverage the object created on the first step.

- Install OPA

OBS.: `NodeRestriction` should be the only admission plugin in the `kube-apiserver`.

```bash
kubectl create -f https://raw.githubusercontent.com/killer-sh/cks-course-environment/master/course-content/opa/gatekeeper.yaml
```

Official install guide [here](https://open-policy-agent.github.io/gatekeeper/website/docs/install/).

Check:

```bash
kubectl get crd

NAME                                                 CREATED AT
configs.config.gatekeeper.sh                         2021-10-20T12:17:03Z
constraintpodstatuses.status.gatekeeper.sh           2021-10-20T12:17:03Z
constrainttemplatepodstatuses.status.gatekeeper.sh   2021-10-20T12:17:03Z
constrainttemplates.templates.gatekeeper.sh          2021-10-20T12:17:04Z
```

Example 1: restrict pods to be deployed.

Create a `ConstraintTemplate` object. When applied, this will generate a new object in the cluster which can be used to specify which kind of resources will be subject to the specified action.

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8salwaysdeny
spec:
  crd:
    spec:
      names:
        kind: K8sAlwaysDeny
      validation:
        openAPIV3Schema:
          properties:
            message:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8salwaysdeny
        violation[{"msg": msg}] {
          1 > 0
          msg := input.parameters.message
        }
```

Create the `K8sAlwaysDeny` object, informing to the `ConstraintTemplate` that the policy will be applied to `pods`.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAlwaysDeny
metadata:
  name: pod-always-deny
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    message: "No POD shall pass!"
```

After applying both templates, the below resources should be present in the cluster.

```bash
kubectl get constraint,constrainttemplate

NAME                                                      AGE
k8salwaysdeny.constraints.gatekeeper.sh/pod-always-deny   37s
NAME                                                       AGE
constrainttemplate.templates.gatekeeper.sh/k8salwaysdeny   87m
```

Trying to launch a new pod, an exception message must be shown and the pod shouldn't be created.

```bash
kubectl run multi --image=mstelles/multi -- sleep 1d

Error from server ([pod-always-deny] ACCESS DENIED!): admission webhook "validation.gatekeeper.sh" denied the request: [pod-always-deny] No POD shall pass!
```

Example 2: Policy to enforce that new `NameSpaces` should have the label "CKS". On this example the violation section uses json formated output fields to create the `provided` field ('.metadata.labels["cks"]') to evaluate the expression. And then, uses the input from the pod manifest as `required`.

Create the `ContraintTemplate`.

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          properties:
            labels:
              type: array
              items: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("you must provide labels: %v", [missing])
        } 
```

Create the `k8srequiredlabels` object, defined in the `ConstraintTemplate`, to be applied to `Namespaces`.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-cks
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["cks"]
```

Check the created resources.

```bash
kubectl get constrainttemplate,constraint

NAME                                                           AGE
constrainttemplate.templates.gatekeeper.sh/k8srequiredlabels   2m51s

NAME                                                           AGE
k8srequiredlabels.constraints.gatekeeper.sh/ns-must-have-cks   8s
```

When creating a `Namespace` without a label with name `CKS`, it shall fail showing the previously configured exception message.

```bash
kubectl create ns whatever

Error from server ([ns-must-have-cks] you must provide labels: {"cks"}): admission webhook "validation.gatekeeper.sh" denied the request: [ns-must-have-cks] you must provide labels: {"cks"}
```

To be able to create the `Namespace`, just add a label named `CKS` with any value to the manifest.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: whatever
  labels:
    cks: bah
```

Example 3: Enforce that a `deployment` should have a minimum value for the replicas.

Create the `ConstraintTemplate`.

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sminreplicacount
spec:
  crd:
    spec:
      names:
        kind: K8sMinReplicaCount
      validation:
        openAPIV3Schema:
          properties:
            min:
              type: integer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sminreplicacount
        violation[{"msg": msg, "details": {"missing_replicas": missing}}] {
          provided := input.review.object.spec.replicas
          required := input.parameters.min
          missing := required - provided
          missing > 0
          msg := sprintf("you must provide %v more replicas", [missing])
        }
```

Create the `K8sMinReplicaCount` object.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sMinReplicaCount
metadata:
  name: deployment-must-have-min-replicas
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    min: 5
```

Sample error message when trying to create a deployment with one replica:

```bash
Error from server ([deployment-must-have-min-replicas] you must provide 4 more replicas): error when creating "deploy.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [deployment-must-have-min-replicas] you must provide 4 more replicas
```

Example 4: Enforce usage of images from a certain registry.

Template:

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8strustedimages
spec:
  crd:
    spec:
      names:
        kind: K8sTrustedImages
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8strustedimages

        violation[{"msg": msg}] {
          image := input.review.object.spec.containers[_].image
          not startswith(image, "docker.io/")
          not startswith(image, "k8s.gcr.io/")
          msg := "not trusted image!"
        }
```

Constraint

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sTrustedImages
metadata:
  name: pod-trusted-images
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```



#### Using OPA to perform conftests

The OPA project created a container image that can be used to perform conf tests on manifest files and check if the deployments will be in conformity with the policies.

Below an example using the below deployment manifest and rego policy document.

- Deployment manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test
  name: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      securityContext:
        runAsNonRoot: true
      containers:
        - image: httpd
          name: httpd
```

- Policy (added on `./project` directory):

```yaml
package main

deny[msg] {
  input.kind = "Deployment"
  not input.spec.template.spec.securityContext.runAsNonRoot = true
  msg = "Containers must not run as root"
}

deny[msg] {
  input.kind = "Deployment"
  not input.spec.selector.matchLabels.app
  msg = "Containers must provide app label for pod selectors"
}
```

- Test:

```bash
docker run --rm -v $(pwd):/project openpolicyagent/conftest test deploy.yaml
```

This same docker image can be used to perform conftests on Dockerfiles. The below example will output an alarm message when the image is based on `ubuntu` and when some command defined in the `commands.rego` file is being used in the Dockerfile.

- Dockerfile

```dockerfile
FROM ubuntu
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y golang-go
COPY app.go .
RUN go build app.go
CMD ["./app"]
```

- Policy files (added on `./project` directory):

```yaml
#base.rego
package main

denylist = [
  "ubuntu"
]

deny[msg] {
  input[i].Cmd == "from"
  val := input[i].Value
  contains(val[i], denylist[_])

  msg = sprintf("unallowed image found %s", [val])
}
```

```yaml
#commands.rego
package commands

denylist = [
  "apk",
  "apt",
  "pip",
  "curl",
  "wget",
]

deny[msg] {
  input[i].Cmd == "run"
  val := input[i].Value
  contains(val[_], denylist[_])

  msg = sprintf("unallowed commands found %s", [val])
}
```

- Test:

```bash
docker run --rm -v $(pwd):/project openpolicyagent/conftest test Dockerfile --all-namespaces
```

- Really useful link with samples and a testing tool to evaluate the policies: https://play.openpolicyagent.org/



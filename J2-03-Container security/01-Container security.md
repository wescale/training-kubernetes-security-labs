# Container security

In this exercise, we will focus on container security regarding container images as well as Kubernetes deployments using the Trivy tool.

## Container image

Inside the current directory, you have a set of files used to build a container image for a sample application.

The goal is to make sure that we build the safest image possible to prevent attacks when it will be deployed on a Kubernetes cluster.

First, inspect the given files to make your self an idea of what you are dealing with.

Build the docker image using the `podman build` command.

Then using the `trivy` CLI inspect both the docker image and the local filesystem to fix vulnerabilities (at least HIGH and CRITICAL) and ensure best practices are applied.

*Hint: the `trivy image` and `trivy fs` commands might help you with that*


## Trivy operator

Since we now have a secure image, we now need to make sure that there is not any vulnerability inside our Kubernetes cluster.

To do that, we will make use of the Trivy Operator. Install it on the cluster using the Helm chart.

```sh
helm repo add aqua https://aquasecurity.github.io/helm-charts/

helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system \
  --create-namespace
```

Look at the pods created and inspect the associated CRDs in the aquasecurity.github.io APIGroup

Now deploy the sample application we built in the previous section (The docker image has been pushed to dockerhub to ease the process)

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: my-sample-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-sample-app
  namespace: my-sample-ns
  labels:
    app: my-sample-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-sample-app
  template:
    metadata:
      labels:
        app: my-sample-app
    spec:
      containers:
      - name: app
        image: docker.io/rdavaze/my-sample-app:1.0.0
        ports:
        - containerPort: 8080
        securityContext:
          privileged: true
```

Verify that the VulnerabilityReport regarding our container image does not contain any critical or high vulnerability as expected.

Then look at the ConfigAuditReport and fix as much misconfigurations as possible by starting by the highest severities.
There might be a false positive with the KSV116 rule, ignore it.

**Note that each time you reconfigure the Deployment a new ConfigAuditReport is automatically created**

## Cleanup

```sh
kubectl delete ns my-sample-ns
```

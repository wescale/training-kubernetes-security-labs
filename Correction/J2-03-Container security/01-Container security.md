# Container security

In this exercise, we will focus on container security regarding container images as well as Kubernetes deployments using the Trivy tool.

## Container image

Inside the current directory, you have a set of files used to build a container image for a sample application.

The goal is to make sure that we build the safest image possible to prevent attacks when it will be deployed on a Kubernetes cluster.

First, inspect the given files to make your self an idea of what you are dealing with.

Build the docker image using the `podman build` command.

Then using the `trivy` CLI inspect both the docker image and the local filesystem to fix vulnerabilities (at least HIGH and CRITICAL) and ensure best practices are applied.

*Hint: the `trivy image` and `trivy fs` commands might help you with that*

Commands to use :

```sh
trivy --security-checks vuln,secret,config ./

podman build -t my-sample-app .
trivy image my-sample-app
```

Fixes:

- In the Dockerfile
  - Change base image to a more recent one : `python:3.12-alpine`
  - Add instructions to upgrade openssl and pip : `apk add --upgrade openssl && pip3 install --upgrade pip`
  - Add a custom user and group : `adduser -u 10001 -D myuser && addgroup --gid 20000 mygroup && addgroup myuser mygroup`
  - Add a USER instruction to prevent image running with root user : `USER 1001`
  - Add healthcheck instruction : `HEALTHCHECK CMD curl --fail http://localhost:8080`
- In the requirements file
  - Update the Flask dependency : `Flask==3.0.0`


## Trivy operator

Since we now have a secure image, we now need to make sure that there is not any vulnerability inside our Kubernetes cluster.

To do that, we will make use of the Trivy Operator. Install it on the cluster using the Helm chart.

```sh
helm repo add aqua https://aquasecurity.github.io/helm-charts/

helm install trivy-operator aqua/trivy-operator \
  --namespace trivy-system \
  --set trivyOperator.scanJobTolerations[0].key="node-role.kubernetes.io/control-plane"  \
  --set trivyOperator.scanJobTolerations[0].operator="Equal" \
  --set trivyOperator.scanJobTolerations[0].value="" \
  --set trivyOperator.scanJobTolerations[0].effect="NoSchedule" \
  --create-namespace
```

Look at the pods created and inspect the associated CRDs in the aquasecurity.github.io APIGroup

> kubectl get po -n trivy-system
> kubectl get crd | grep aquasecurity.github.io
> 2 important resources : vulnerabilityreports and configauditreports


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

> kubectl describe vuln -n my-sample-ns

Then look at the ConfigAuditReport and fix as much misconfigurations as possible by starting by the highest severities.
There might be a false positive with the KSV116 rule, ignore it.

> kubectl describe configaudit -n my-sample-ns

**Note that each time you reconfigure the Deployment a new ConfigAuditReport is automatically created**

Complete manifest:

```yaml
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
      securityContext:
        fsGroup: 20000
        runAsUser: 10001
        runAsGroup: 20000
        supplementalGroups: [20000]
      containers:
      - name: app
        image: docker.io/rdavaze/my-sample-app:1.0.0
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          runAsNonRoot: true
          runAsUser: 10001
          runAsGroup: 20000
          readOnlyRootFilesystem: true
          seccompProfile:
            type: RuntimeDefault
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
          limits:
            cpu: 400m
            memory: 256Mi
```

## Cleanup

```sh
kubectl delete ns my-sample-ns
```

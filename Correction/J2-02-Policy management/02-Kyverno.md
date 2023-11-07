# Kyverno

## Install Kyverno

```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

Verify that Kyverno pods are running and check the CRDs that have been created

## Basic policies

Kyverno has a big amount of policies already written. You can look at them here: https://kyverno.io/policies/

To ease the implementation of generic policies, an additional Helm chart is available. Install it:

```sh
helm install kyverno-policies kyverno/kyverno-policies -n kyverno
```

Look at the ClusterPolicy resources installed and guess what they are doing.

Try to violate the `hostPath` policy by deploying a sample app mounting the root directory of the host beneath in a dedicated `test-kyverno` namespace

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-path-violation-pod
  namespace: test-kyverno
spec:
  containers:
  - image: k8s.gcr.io/pause:3.1
    name: pause
    volumeMounts:
    - mountPath: /host
      name: host
  volumes:
  - name: host
    hostPath:
      path: /
      type: Directory
```

Deploy the pod. Is it running ? Why ? Describe the pod and look at the events

> Yes because the clusterpolicy `validationFailureAction` property is set to audit

What can we do to make sure that the pod will not be created ? Do the necessary changes and try to recreate the pod.

> Edit the clusterpolicy to set it to enforce

```sh
kubectl patch clusterpolicy disallow-host-path --type=json -p '[{"op": "replace", "path": "/spec/validationFailureAction", "value": "enforce"}]'
```

## Custom policy

Create a custom ClusterPolicy resource that requires more than 1 replica on Deployments with the `high-availability: true` label

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-deployment-has-multiple-replicas
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: deployment-has-multiple-replicas
      match:
        any:
        - resources:
            kinds:
            - Deployment
            selector:
              matchLabels:
                high-avaibility: "true"
      validate:
        message: "Deployments should have more than one replica to ensure availability."
        pattern:
          spec:
            replicas: ">1"
```

Try to create the following Deployment and see if it is created.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-single
  namespace: test-kyverno
  labels:
    high-avaibility: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:alpine3.18
        name: nginx
        ports:
        - containerPort: 8080
```

Fix the Deployment according to the ClusterPolicy you created and redeploy it.

## Cleanup

```sh
kubectl delete clusterpolicy --all
```

# Lab 05 - Runtime Security

## Runtime configuration

### Install Runtime runsc on worker-0

Script install/build containerd-shim-runsc-v1

```sh
#!/usr/bin/env bash

set -e

ARCH=$(uname -m)

URL=https://storage.googleapis.com/gvisor/releases/release/latest/${ARCH}

wget ${URL}/runsc ${URL}/runsc.sha512 ${URL}/containerd-shim-runsc-v1 ${URL}/containerd-shim-runsc-v1.sha512

sha512sum -c runsc.sha512 -c containerd-shim-runsc-v1.sha512

rm -f *.sha512

sudo chmod a+rx runsc containerd-shim-runsc-v1

sudo mv runsc containerd-shim-runsc-v1 /usr/local/bin

```

Edit and add runsc runtime to containerd

```conf
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
    runtime_type = "io.containerd.runsc.v1"
```

Restart containerd runtime

### Run a pod with gVisor Runtime

Create a runtime class gVisor

```yaml
apiVersion: node.k8s.io/v1  
kind: RuntimeClass
metadata:
  name: gvisor  
handler: runsc
```

Start a pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
spec:
  containers:
  - image: ubuntu
    command:
      - sleep
      - infinity
    name: ubuntu
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ubuntu-gvisor
spec:
  selector:
      matchLabels:
        name: ubuntu-gvisor
  template:
    metadata:
      labels:
        name: ubuntu-gvisor
    spec:
      runtimeClassName: gvisor
      containers:
      - image: ubuntu
        command:
          - sleep
          - infinity
        name: ubuntu
```

Checks

```sh
kubectl exec -it ubuntu -- bash
```

Checks

```sh
kubectl get po
kubectl exec -it ubuntu-gvisor-xxxxx -- bash
```

## Runtime checks

### Install Falco

Download helm chart

```sh
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

Install falco

```sh
kubectl create namespace falco

helm install falco -n falco --set driver.kind=ebpf --set tty=true falcosecurity/falco
```

Simulate suspicious activity

Generate a pod and activity

```sh
kubectl run ubuntu --image ubuntu -- bash -c "sleep infinity"
kubectl exec -it ubuntu -- bash -c "uptime"
```

Check the ouput

```sh
kubectl logs -l app.kubernetes.io/name=falco -n falco -c falco | grep Notice
```

### Create custom rule

Custom rule on ssh connection

```yaml
customRules:
  rules-ssh.yaml: |-
    - rule: SSH attempt by non-allowlisted client
      desc: SSH attempt by non-allowlisted client
      condition: >
        evt.type = accept and
        evt.dir = < and
        proc.name = sshd and
        fd.cip != "10.124.0.2"
      output: >
        SSH attempt by non-allowlisted client
        (client=%fd.cip
        pcmdline=%proc.pcmdline
        container_id=%container.id
        image=%container.image.repository)
      priority: Notice
      tags: [network, mitre_remote_service, mitre_discovery]
```

Apply the rule

```sh
helm upgrade falco -n falco -f custom-rules.yaml --set driver.kind=ebpf --set tty=true falcosecurity/falco
```

SSH into a node

Check the ouput

```sh
kubectl logs -l app.kubernetes.io/name=falco -n falco -c falco | grep Notice
```

> https://falco.org/docs/reference/rules/default-rules/

### Adds

<details>

[About gvisor](https://github.com/falcosecurity/charts/tree/master/charts/falco#about-gvisor)

Example: running Falco on GKE, with or without gVisor-enabled pods
If you use GKE with k8s version at least 1.24.4-gke.1800 or 1.25.0-gke.200 with gVisor sandboxed pods, you can install a Falco instance to monitor them with, e.g.:

```sh
helm install falco-gvisor falcosecurity/falco -f https://raw.githubusercontent.com/falcosecurity/charts/master/falco/values-gvisor-gke.yaml --namespace falco-gvisor --create-namespace
```

Note that the instance of Falco above will only monitor gVisor sandboxed workloads on gVisor-enabled node pools. If you also need to monitor regular workloads on regular node pools you can use the eBPF driver as usual:

```sh
helm install falco falcosecurity/falco --set driver.kind=ebpf --namespace falco --create-namespace
```

The two instances of Falco will operate independently and can be installed, uninstalled or configured as needed. If you were already monitoring your regular node pools with eBPF you don't need to reinstall it.

</details>


### Falco alternative (optionnal)

Install Tracee

```sh
helm repo add aqua https://aquasecurity.github.io/helm-charts/
helm repo update
helm install tracee aqua/tracee --namespace tracee --create-namespace
```

Simulate suspicious activity

Generate a pod and activity

```sh
kubectl run tracee-tester --image=aquasec/tracee-tester -- TRC-105
```

Check the ouput

```sh
kubectl logs -l app.kubernetes.io/name=tracee -n tracee -c tracee | grep fileless_execution 

kubectl logs -l app.kubernetes.io/name=tracee -n tracee -c tracee | grep -i ssh 
```

## Clean up

```sh
kubectl delete po ubuntu --ignore-not-found=true
kubectl delete ds ubuntu-gvisor --ignore-not-found=true

kubectl delete ns tracee --ignore-not-found=true
kubectl delete ns falco --ignore-not-found=true
```
# 01-Pod Security Admission (PSA)

## Configure the built-in Admission Controller

### Create a yaml config file with this content

```sh
sudo mkdir /etc/kubernetes/custom-config
sudo vi /etc/kubernetes/custom-config/psa.yaml
```

```yaml

apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1 # see compatibility note
    kind: PodSecurityConfiguration
    # Defaults applied when a mode label is not set.
    #
    # Level label values must be one of:
    # - "privileged" (default)
    # - "baseline"
    # - "restricted"
    #
    # Version label values must be one of:
    # - "latest" (default) 
    # - specific version like "v1.28"
    defaults:
      enforce: "privileged"
      enforce-version: "latest"
      audit: "privileged"
      audit-version: "latest"
      warn: "privileged"
      warn-version: "latest"
    exemptions:
      # Array of authenticated usernames to exempt.
      usernames: []
      # Array of runtime class names to exempt.
      runtimeClasses: []
      # Array of namespaces to exempt.
      namespaces: []

```

### target this file to the kube-apiserver config and manage required volume

add this content to ```/etc/kubernetes/manifests/kube-apiserver.yaml```

```sh

# command line option
    - --admission-control-config-file=/etc/kubernetes/custom-config/psa.yaml

# volumeMounts:
    - mountPath: /etc/kubernetes/custom-config
      name: custom-config
      readOnly: true

# volumes:
  - hostPath:
      path: /etc/kubernetes/custom-config
      type: DirectoryOrCreate
    name: custom-config
```

```sh

# check 
sudo crictl ps -a | grep kube-apiserver
sudo crictl logs -f <kube-apiserver-container-id>

    # if you have this error, there is an issue with the volume
    E1103 08:55:01.328116       1 run.go:74] "command failed" err="failed to initialize admission: failed to read plugin config: unable to read admission control configuration from \"/etc/kubernetes/custom-config/psa.yaml\" [open /etc/kubernetes/custom-config/psa.yaml: no such file or directory]"

```

### check with labeled namespaces

```yaml

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest

    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
EOF


cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: my-restricted-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
EOF

```

```sh

# create a deployment in my-baseline-namespace
kubectl create deployment container-demo --replicas=2 --image=damaumenee/container-demo:1.0 -n my-baseline-namespace

    Warning: would violate PodSecurity "restricted:v1.27": allowPrivilegeEscalation != false (container "container-demo" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "container-demo" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "container-demo" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "container-demo" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
    deployment.apps/container-demo created

# check pod
kubectl get pod -n my-baseline-namespace

# create a deployment in my-restricted-namespace
kubectl create deployment container-demo --replicas=2 --image=damaumenee/container-demo:1.0 -n my-restricted-namespace

    Warning: would violate PodSecurity "restricted:v1.27": allowPrivilegeEscalation != false (container "container-demo" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "container-demo" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "container-demo" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "container-demo" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
    deployment.apps/container-demo created

# check pod
kubectl get pod -n my-restricted-namespace

```
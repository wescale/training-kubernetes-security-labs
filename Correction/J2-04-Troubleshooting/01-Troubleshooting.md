# Troubleshooting

In this exercise, you will be put in a real world situation. A situation will be given to you and your job is to investigate and fix the issues that are present on the cluster.

## Context

You have received a notification from the SOC (Security Operational Center) that some security issues have been discovered on the Kubernetes cluster you operate.

The only indications you have is that issues are coming from the `front-office` and `back-office` namespaces:

- Some Kubernetes permissions are too broad for their usage at namespace and/or cluster level
- Some applications manifests are not configured properly and/or have too broad permissions
- Some sensitive information are exposed
- The cluster is not completly compliant with the CIS benchmark


Your mission, if you accept it (you don't really have the choice) is to investigate and do what is necessary to adress these issues.

Don't hesitate to make use of all that you've seen during the two previous days, it will help.

Commands to find issues:

```sh
kubectl describe vuln -n front-office
kubectl describe configaudit -n front-office
kubectl describe rbacassessmentreports -n front-office

kubectl describe vuln -n back-office
kubectl describe configaudit -n back-office
kubectl describe rbacassessmentreports -n back-office
```

List of things to fix:
> `front-office` role in `front-office` namespace has the permission to retrieve secrets. **Remove the secret permissions on the role**

> `front-office` image uses the tag `latest`. **Find a named tag on dockerhub**

> `back-office` image is an old image and contains vulnerabilities. **Update it to the latest image**

> `back-office` deployment security context can be more secure. **Fix `runAsUser`, `allowPrivilegeEscalation` and `readOnlyRootFilesystem`**

> The `csi clustercompliancereport` resource reports fails. Fix them

```sh
kubectl get compliance cis -o=jsonpath='{.status}' | jq '.summaryReport.controlCheck[] | select(.severity=="CRITICAL")| select(.totalFail>0)'
```

## Falco Check


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

```yaml
customRules:
  rules-container-privileged.yaml: |-
    - list: trusted_images
      items: [docker.io/hashicorp/http-echo, docker.io/calico/node, docker.io/falcosecurity/falco-no-driver, docker.io/calico/node-driver-registrar, docker.io/calico/csi]

    - rule: Launch Privileged Container
      desc: Detect the start of a privileged container. Exceptions are made for known trusted images.
      condition: >
        container
        and container.privileged=true
        and not ( container.image.repository in (trusted_images) or
                  container.image.repository startswith registry.k8s.io/ )
      output: Privileged container
      priority: alert
```

```sh
helm upgrade falco -n falco -f custom-rules.yaml --set driver.kind=ebpf --set tty=true falcosecurity/falco
```
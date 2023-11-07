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

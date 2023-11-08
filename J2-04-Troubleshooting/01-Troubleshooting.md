# Troubleshooting

In this exercise, you will be put in a real world situation. A situation will be given to you and your job is to investigate and fix the issues that are present on the cluster.

## Context

You have received a notification from the SOC (Security Operational Center) that some security issues have been discovered on the Kubernetes cluster you operate.

The only indications you have is that issues are coming from the `front-office` and `back-office` namespaces:

- Some Kubernetes permissions are too broad for their usage at namespace and/or cluster level
- Some applications manifests are not configured properly and/or have too broad permissions
- Some sensitive information are exposed


Your mission, if you accept it (you don't really have the choice) is to investigate and do what is necessary to adress these issues.

Don't hesitate to make use of all that you've seen during the two previous days, it will help.

## Falco Checks

Create a falco rule that reduce alert on `registry.k8s.io` trust your `frontoffice` to be privileged deployment alert


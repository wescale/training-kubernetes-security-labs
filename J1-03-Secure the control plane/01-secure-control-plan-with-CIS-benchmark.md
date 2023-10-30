# Hardening controle plan according CIS

```
[INFO] 1 Control Plane Security Configuration
[INFO] 1.1 Control Plane Node Configuration Files
[WARN] 1.1.9 Ensure that the Container Network Interface file permissions are set to 600 or more restrictive (Manual)
[FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)
[WARN] 1.1.20 Ensure that the Kubernetes PKI certificate file permissions are set to 600 or more restrictive (Manual)
[INFO] 1.2 API Server
[WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false (Manual)
[WARN] 1.2.3 Ensure that the --DenyServiceExternalIPs is set (Manual)
[FAIL] 1.2.5 Ensure that the --kubelet-certificate-authority argument is set as appropriate (Automated)
[WARN] 1.2.9 Ensure that the admission control plugin EventRateLimit is set (Manual)
[WARN] 1.2.11 Ensure that the admission control plugin AlwaysPullImages is set (Manual)
[WARN] 1.2.12 Ensure that the admission control plugin SecurityContextDeny is set if PodSecurityPolicy is not used (Manual)
[FAIL] 1.2.17 Ensure that the --profiling argument is set to false (Automated)
[FAIL] 1.2.18 Ensure that the --audit-log-path argument is set (Automated)
[FAIL] 1.2.19 Ensure that the --audit-log-maxage argument is set to 30 or as appropriate (Automated)
[FAIL] 1.2.20 Ensure that the --audit-log-maxbackup argument is set to 10 or as appropriate (Automated)
[FAIL] 1.2.21 Ensure that the --audit-log-maxsize argument is set to 100 or as appropriate (Automated)
[WARN] 1.2.22 Ensure that the --request-timeout argument is set as appropriate (Manual)
[WARN] 1.2.29 Ensure that the --encryption-provider-config argument is set as appropriate (Manual)
[WARN] 1.2.30 Ensure that encryption providers are appropriately configured (Manual)
[WARN] 1.2.31 Ensure that the API Server only makes use of Strong Cryptographic Ciphers (Manual)
[INFO] 1.3 Controller Manager
[WARN] 1.3.1 Ensure that the --terminated-pod-gc-threshold argument is set as appropriate (Manual)
[FAIL] 1.3.2 Ensure that the --profiling argument is set to false (Automated)
[INFO] 1.4 Scheduler
[FAIL] 1.4.1 Ensure that the --profiling argument is set to false (Automated)
```

## disable profiling

### controller-manager

edit ```/etc/kubernetes/manifests/kube-controller-manager.yaml``` to add ```--profiling=false``` option on the command

```sh

# check kube-bench
sudo kube-bench run --targets=master | grep 1.3.2
  [FAIL] 1.3.2 Ensure that the --profiling argument is set to false (Automated)

sudo vi /etc/kubernetes/manifests/kube-controller-manager.yaml

    - --use-service-account-credentials=true # find this line
    - --profiling=false # add this line



# after saving the file, kube-controller-manager-master restarts automatically   
kubectl get pod -n kube-system 
  NAME                               READY   STATUS    RESTARTS   AGE
  coredns-5d78c9869d-6b6dg           1/1     Running   0          2d23h
  coredns-5d78c9869d-8n4l6           1/1     Running   0          2d23h
  etcd-master-0                      1/1     Running   0          2d23h
  kube-apiserver-master-0            1/1     Running   0          2d23h
  kube-controller-manager-master-0   0/1     Running   0          14s
  kube-proxy-4s7wh                   1/1     Running   0          2d23h
  kube-proxy-6jhsg                   1/1     Running   0          2d23h
  kube-proxy-6xngt                   1/1     Running   0          2d23h
  kube-proxy-j65l7                   1/1     Running   0          42m
  kube-scheduler-master-0            1/1     Running   0          2d23h

# check kube-bench again
sudo kube-bench run --targets=master | grep 1.3.2
  [PASS] 1.3.2 Ensure that the --profiling argument is set to false (Automated)

```
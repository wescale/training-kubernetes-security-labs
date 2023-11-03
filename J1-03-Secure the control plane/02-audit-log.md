# Audit log

```sh
sudo mkdir /etc/kubernetes/custom-config
sudo vim /etc/kubernetes/custom-config/simple-policy.yaml
```

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
```


```sh
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml


```

```yaml
  ...
  #- command:
   # - kube-apiserver

    - --audit-log-path=/var/log/audit.log # fix CIS 1.2.18
    - --audit-log-maxage=30 # fix CIS 1.2.19
    - --audit-log-maxbackup=10 # fix CIS 1.2.20  
    - --audit-log-maxsize=100 # fix CIS 1.2.21
    - --audit-policy-file=/etc/kubernetes/custom-config/simple-policy.yaml #<-- Audit policy file

  
  ...
  
  #volumeMounts:
    - mountPath: /etc/kubernetes/custom-config
      name: custom-config
      readOnly: true
    - mountPath: /var/log/audit.log
      name: audit-log
      readOnly: false
  
  ....

  # volumes:
  - hostPath:
      path: /etc/kubernetes/custom-config
      type: DirectoryOrCreate
    name: custom-config
  - hostPath:
      path: /var/log/audit.log
      type: FileOrCreate
    name: audit-log
```
Why the use of volume is mandatory ?



Tips for troubleshooting kube-apiserver needed
```sh
# check 
sudo crictl ps -a | grep kube-apiserver
sudo crictl logs -f <kube-apiserver-container-id>
```

Check audit logs ```tail -f /var/log/audit.log```
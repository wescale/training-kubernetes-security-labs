# Install K8S HA cluster with kubeadm

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

## prerequesite allready installed for you


<details>
  <summary>prerequesite</summary>
  
```sh

#!/bin/sh
echo "setup hostname=$(hostname) ip=$(hostname -i)"

 

# install K8s pre-req

# 4.  Become root
sudo -i


# SET K8S version
export K8S_MAJOR_VERSION=1.28

# update and upgrade the system.  You may be asked a few questions.  Allow restarts and keep the local version currently installed
sudo apt-get update && sudo apt-get upgrade -y

# 5. install editor if needed
apt-get install -y vim nano



# i.  Use themodprobecommand to load theoverlayand thebr_netfiltermodules.
modprobe overlay
modprobe br_netfilter

# ii. setup /etc/sysctl.d/99-kubernetes-cri.conf 
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# iii.  Use thesysctlcommand to apply the config file.
sysctl --system

#-----------------------------------------------------------------------------
# 6. install ContainerD

# 6.1 install containerd.io with and apt-get (isntall runc too, but does not contain CNI plugins).

# https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-2-from-apt-get-or-dnf
# Add Docker's official GPG key:
apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```


```sh
sudo apt-get update

# install containerd.io with and apt-get
apt-get install containerd.io

# 6.2 runc is include in containerd.io package

# 6.3 CNI plugin : https://github.com/containerd/containerd/blob/main/docs/getting-started.md#step-3-installing-cni-plugins
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz

# 6.4 Configuring the systemd cgroup driver
# https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd
cat /vagrant/vagrant-provisionning/containerd/config.toml > /etc/containerd/config.toml
systemctl restart containerd


# 7. Add a new repo for kubernetes. You could also download a tar file or use code from GitHub. Create the file and add anentry for the main repo for your distribution.  We are using theUbuntu 20.04 but the kubernetes-xenial repo of the software, also include the key wordmain. Note there are four sections to the entry.
 echo \
 "deb  http://apt.kubernetes.io/  kubernetes-xenial  main" \
 | tee -a /etc/apt/sources.list.d/kubernetes.list


# 8.  Add a GPG key for the packages. The command spans three lines. You can omit the backslash when you type. The OK is the expected output, not part of the command.
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

# 9.  Update with the new repo declared, which will download updated repo information.
apt-get update

# 10.  Install the software. There are regular releases, the newest of which can be used by omitting the equal sign and version information on the command line. Historically new versions have lots of changes and a good chance of a bug or five. As a result we will hold the software at the recent but stable version we install. 
# In a later lab we will update the cluster to a newer version. 
K8S_EXACT_VERSION=$(apt-cache madison kubectl | grep $K8S_MAJOR_VERSION | head -1 | awk '{print $3}')

apt-get install -y kubeadm=$K8S_EXACT_VERSION kubelet=$K8S_EXACT_VERSION kubectl=$K8S_EXACT_VERSION

apt-mark hold kubelet kubeadm kubectl


```

</details>


## calico prerequisite

https://docs.tigera.io/calico/latest/operations/troubleshoot/troubleshooting#configure-networkmanager

```sh



# https://docs.tigera.io/calico/latest/operations/troubleshoot/troubleshooting#configure-networkmanager
# [keyfile]
# unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
sudo mkdir -p /etc/NetworkManager/conf.d/

sudo bash -c 'cat > /etc/NetworkManager/conf.d/calico.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
EOF'


```

## init the first cluster master


```sh

# connect to master-0
ssh master-0

# optionnal :
# sudo kubeadm config images pull
# this config depends on CNI, we use Calico --pod-network-cidr=192.168.0.0/16 
# The --control-plane-endpoint flag should be set to the address or DNS and port of the load balancer.
# THE DNS is k8s-api.k8s-sec-<YOUR-UNIQUE-NUMBER>.wescaletraining.fr
# The --upload-certs flag is used to upload the certificates that should be shared across all the control-plane instances to the cluster.

# You MUST change <YOUR-UNIQUE-NUMBER> before !!
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint "k8s-api.k8s-sec-<YOUR-UNIQUE-NUMBER>.wescaletraining.fr:6443" --upload-certs

# SAVE the outputs wich looks similar to :
    You can now join any number of control-plane node by running the following command on each as a root:
        kubeadm join k8s-api.k8s-sec-0.wescaletraining.fr:6443:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07

    Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
    As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

    Then you can join any number of worker nodes by running the following on each as root:
        kubeadm join k8s-api.k8s-sec-0.wescaletraining.fr:6443:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866




# allow k8s api acces from current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


# auto completion
echo "source <(kubectl completion bash)" >> $HOME/.bashrc
echo "alias k=kubectl" >> $HOME/.bashrc
echo "complete -o default -F __start_kubectl k" >> $HOME/.bashrc
source $HOME/.bashrc


    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 10.124.1.7:6443 --token 7l7l2c.d4rftkr3i6fro1fh \
            --discovery-token-ca-cert-hash sha256:0ca8a6bbc11fe3a91f8fcd21076153fc6247395a098134f934d5096a5d896bcd
            
```

Save the kubeadm join command with the --token and --discovery-token-ca-cert-hash

### install Calico


https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart#install-calico


```sh

# 1. Install the Tigera Calico operator and custom resource definitions.
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/tigera-operator.yaml

# 2. Install Calico by creating the necessary custom resource.
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/custom-resources.yaml

# 3. Confirm that all of the pods are running with the following command. Wait until each pod has the STATUS of Running.
watch kubectl get pods -n calico-system
    NAME                                       READY   STATUS    RESTARTS   AGE
    calico-kube-controllers-8595fcb4f6-dhn64   1/1     Running   0          2m6s
    calico-node-92z5q                          1/1     Running   0          2m6s
    calico-typha-75895fdc8-bvsnd               1/1     Running   0          2m7s
    csi-node-driver-c4pfn                      2/2     Running   0          2m6s

```

## Add master


Use the command displayed during init phase, with your own tokens !

```sh
    kubeadm join k8s-api.k8s-sec-0.wescaletraining.fr:6443:6443 --token 9vr73a.a8uxyaju799qwdjv \ 
      --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 \
      --control-plane \
      --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
```

        

## Add worker node

Use the command displayed during init phase, with your own tokens !

```sh
    sudo kubeadm join 10.124.1.7:6443 --token 7l7l2c.d4rftkr3i6fro1fh \
            --discovery-token-ca-cert-hash sha256:0ca8a6bbc11fe3a91f8fcd21076153fc6247395a098134f934d5096a5d896bcd

```

By default join token is only valid for 24 hours

Error message if invalid token

```sh
sudo kubeadm join 10.124.1.7:6443 --token 7l7l2c.d4rftkr3i6fro1fh \
>             --discovery-token-ca-cert-hash sha256:0ca8a6bbc11fe3a91f8fcd21076153fc6247395a098134f934d5096a5d896bcd
[preflight] Running pre-flight checks
error execution phase preflight: couldn't validate the identity of the API Server: could not find a JWS signature in the cluster-info ConfigMap for token ID "7l7l2c"
```

```sh
kubeadm token create --print-join-command

kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
12ytiu.8d4iqxdn92wt07gp   23h         2023-10-31T07:46:37Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
```
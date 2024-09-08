# Install K8S HA cluster with kubeadm

<https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/>

## Prerequisites already installed for you

<details>
  <summary>Prerequisites</summary>

```sh
#!/bin/bash

set -euo pipefail

export INSTALL_K8S_VERSION="${k8s_version}" #exemple : 1.30

# 1. install K8s pre-req

# update and upgrade the system.  You may be asked a few questions.  Allow restarts and keep the local version currently installed
apt-get update

# install editor if needed
apt-get install -y vim nano jq

# i. Use themodprobecommand to load theoverlayand thebr_netfiltermodules.
modprobe br_netfilter

# ii. setup /etc/sysctl.d/99-kubernetes-cri.conf
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.ipv4.ip_forward                 = 1
EOF

# iii.  Use thesysctlcommand to apply the config file.
sysctl --system


#-----------------------------------------------------------------------------
# 2. install ContainerD

# 2.1 install containerd.io with and apt-get (isntall runc too, but does not contain CNI plugins).

# https://github.com/containerd/containerd/blob/main/docs/getting-started.md#option-2-from-apt-get-or-dnf
# Add Docker's official GPG key:
apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# install containerd.io with and apt-get
apt-get install -y containerd.io

# 2.3 CNI plugin : https://github.com/containerd/containerd/blob/main/docs/getting-started.md#step-3-installing-cni-plugins
wget https://github.com/containernetworking/plugins/releases/download/v1.5.1/cni-plugins-linux-amd64-v1.5.1.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.1.tgz


# 2.4 Configuring the systemd cgroup driver
# https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd
cat <<EOF > /etc/containerd/config.toml
#   Copyright 2018-2022 Docker Inc.

#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at

#       http://www.apache.org/licenses/LICENSE-2.0

#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

disabled_plugins = []

# W1022 11:03:27.554707   35125 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.

version = 2

[plugins."io.containerd.runtime.v1.linux"]

shim_debug = true

[plugins."io.containerd.grpc.v1.cri"]
sandbox_image = "registry.k8s.io/pause:3.9"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]

runtime_type = "io.containerd.runc.v2"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true


#root = "/var/lib/containerd"
#state = "/run/containerd"
#subreaper = true
#oom_score = 0

#[grpc]
#  address = "/run/containerd/containerd.sock"
#  uid = 0
#  gid = 0

#[debug]
#  address = "/run/containerd/debug.sock"
#  uid = 0
#  gid = 0
#  level = "info"
EOF

systemctl restart containerd


#-----------------------------------------------------------------------------
# 3. install kube tools for install

# Add the Kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v$INSTALL_K8S_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$INSTALL_K8S_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list

# 3.2  Update with the new repo declared, which will download updated repo information.
apt-get update

# 3.3  Install the software. There are regular releases, the newest of which can be used by omitting the equal sign and version information on the command line. Historically new versions have lots of changes and a good chance of a bug or five. As a result we will hold the software at the recent but stable version we install.
# pick the second to last version because In a later lab we will update the cluster to a newer version.
# include crictl
K8S_EXACT_VERSION=$(apt-cache madison kubectl | grep $INSTALL_K8S_VERSION | head -2 | sort | head -1 | awk '{print $3}')

apt-get install -y kubeadm=$K8S_EXACT_VERSION kubelet=$K8S_EXACT_VERSION kubectl=$K8S_EXACT_VERSION

apt-mark hold kubelet kubeadm kubectl

# 4. Install kube-bench
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.8.0/kube-bench_0.8.0_linux_amd64.deb -o kube-bench_0.8.0_linux_amd64.deb
sudo dpkg -i ./kube-bench_0.8.0_linux_amd64.deb > /dev/null

# 5. Install Trivy
wget https://github.com/aquasecurity/trivy/releases/download/v0.54.1/trivy_0.54.1_Linux-64bit.deb
sudo dpkg -i trivy_0.54.1_Linux-64bit.deb  > /dev/null
```
</details>

## Init the first cluster master

```sh
# connect to master-0
ssh master-0

# /!\ You MUST change <YOUR-UNIQUE-NUMBER> before !!
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint "k8s-api.k8s-sec-<YOUR-UNIQUE-NUMBER>.wescaletraining.fr:6443" --upload-certs
# /!\ SAVE THE TWO OUTPUTED COMMANDS
# They will be used by other nodes to join the cluster

# Allow k8s api access from current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# You should now see your cluster with a single master node
kubectl get nodes


# Add auto completion
echo "source <(kubectl completion bash)" >> $HOME/.bashrc
echo "alias k=kubectl" >> $HOME/.bashrc
echo "complete -o default -F __start_kubectl k" >> $HOME/.bashrc
source $HOME/.bashrc
```

### Install CNI : Cilium




```sh


CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64

curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz

# Install Cilium into the Kubernetes cluster pointed to by your current kubectl context
cilium install --version 1.16.1

# validate that Cilium has been properly installed
cilium status --wait

# validate that your cluster has proper network connectivity
# cilium connectivity test

```


## Master nodes

**Execute these commands on master-1 and master-2**

Use the command displayed during init phase, with your own tokens !

```sh
    kubeadm join k8s-api.k8s-sec-0.wescaletraining.fr:6443 --token 9vr73a.a8uxyaju799qwdjv \
      --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 \
      --control-plane \
      --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
```

## Worker nodes

**Execute these commands on worker-0, worker-1 and worker-2**

Use the command displayed during init phase, with your own tokens !

```sh
    sudo kubeadm join 10.124.1.7:6443 --token 7l7l2c.d4rftkr3i6fro1fh \
            --discovery-token-ca-cert-hash sha256:0ca8a6bbc11fe3a91f8fcd21076153fc6247395a098134f934d5096a5d896bcd

```

**/!\ By default join token is only valid for 24 hours. If it has expired run the following command from the master-0 to generate a new one**

```sh
kubeadm token create --print-join-command

kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
12ytiu.8d4iqxdn92wt07gp   23h         2023-10-31T07:46:37Z   authentication,signing   <none>                                                     system:bootstrappers:kubeadm:default-node-token
```

## Retrieve kubeconfig

Go back to the bastion and retrieve the cluster kubeconfig file

```sh
mkdir -p ~/.kube/config

scp -F provided_ssh_config master-0:/home/training/.kube/config ~/.kube
```

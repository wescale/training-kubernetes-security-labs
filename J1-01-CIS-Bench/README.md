# Lab 01 - OS CIS Benchmark

## Connect to SSH


At this stage, you get a cluster with the control plane initialized with RKE2, but the cluster has no worker.

Ensure the control plane is OK by connecting to the bastion instance.

For that, download the [private SSH key](https://raw.githubusercontent.com/WeScale/k8s-advanced-training/master/resources/kubernetes-formation) on you personal laptop and start an ssh agent to add the key and connect to the instance:

```sh
chmod 400 kubernetes-formation

eval "$(ssh-agent -s)"
ssh-add kubernetes-formation
# Ensure the key is present
ssh-add -L 
# SSH
ssh -A -F provided_ssh_config bastion
```

When bastion is ok, you can now navigate on all instances.

Go to `cis` instance

```sh
ssh -F provided_ssh_config cis
```

## Check CIS Benchmark - OS

`cis` instance is a Google recommanded OS.
CIS benchmark are built-in service installed.

The level 1 is already active and have a periodical check.

Check services

```sh
systemctl status cis-level1
systemctl status cis-level2
```

Found and show the results

<details>

```sh
cat /var/lib/google/cis_scanner_scan_result.textproto
```

</details>

Now `exit` and return to the `bastion` host

```sh
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    certbot
```

Install Docker :
```bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io

sudo usermod -aG docker training # Relaunch the ssh session

sudo docker run -d -p 80:80 -v /var/www/html:/usr/share/nginx/html -n nginx nginx 

echo "open: http://bastion.k8s-sec-wsc-k8s-security-0.wescaletraining.fr"
```

Then launch CIS bench mark
```sh
unzip CIS-CAT-Lite-Assessor-v4.34.0.zip
cd Assessor 
sudo sh ./Assessor-CLI.sh -i -rd /var/www/html/ -nts -rp index
```

<details>

`sh`: This is the shell interpreter used to execute the script.

`./Assessor-CLI.sh`: This is the path to the CIS-CAT Pro Assessor command-line interface (CLI) script.

`-i`: This option typically indicates an "interactive" mode, where the tool may prompt the user for input or display interactive dialogs.

`-rd` /var/www/html/: This option specifies the root directory (/var/www/html/) to assess. This is where CIS-CAT Pro Assessor will look for configuration files to evaluate against the selected benchmark.

`-nts`: This option likely stands for "No Timestamps," and it might instruct the tool not to include timestamps in its output.

`-rp index`: This option allows you to specify a custom report prefix. When you use this option, the tool will generate a report with the specified prefix in the filename.

</details>

Run the test in interactive mode and use below settings:
- Benchmarks/Data-Stream Collections: CIS Ubuntu Linux 20.04 LTS Benchmark v2.0.0,
- Profile : Level 1 - Server

## Clean up

```sh
sudo docker container rm -f nginx
sudo apt-get remove -y docker-ce docker-ce-cli containerd.io
```
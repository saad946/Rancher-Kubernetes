# Introduction to Rancher: On-prem Kubernetes

# Hyper-V : Prepare our infrastructure

In this demo, we will use Hyper-V to create our infrastructure.  </br>
For on-premise, many companies use either Hyper-V, VMWare Vsphere and other technologies to create virtual infrastructure on bare metal. </br>

Few points to note here:

* Benefit of Virtual infrastructure is that it's immutable
  a) We can add and throw away virtual machines at will.
  b) This makes maintenance easier as we can roll updated virtual machines instead of 
     patching existing machines and turning them to long-living snowflakes.
  c) Reduce lifespan of machines

* Bare Metal provides the compute. 
  a) We don't want Kubernetes directly on bare metal as we want machines to be immutable.
  b) This goes back to the previous point on immutability.

* Every virtual machine needs to be able to reach each other on the network
  a) This is a kubernetes networking requirements that all nodes can communicate with one another

# Hyper-V : Create our network

In order for us to create virtual machines all on the same network, I am going to create a virtual switch in Hyper-v </br>
Open Powershell in administrator

```
# get our network adapter where all virtual machines will run on
# grab the name we want to use

Get-Module -ListAvailable -Name Hyper-V
Import-Module Hyper-V 
or
Import-Module -Name Hyper-V -RequiredVersion 2.0.0.0
$ethernet = Get-NetAdapter -Name "Ethernet 2"
New-VMSwitch -Name "virtual-network" -NetAdapterName $ethernet.Name -AllowManagementOS $true -Notes "shared virtual network interface"


# Hyper-V : Create our machines

We firstly need harddrives for every VM. </br>
Let's create three:

```
mkdir c:\temp\vms\linux-0\
mkdir c:\temp\vms\linux-1\
mkdir c:\temp\vms\linux-2\

New-VHD -Path c:\temp\vms\linux-0\linux-0.vhdx -SizeBytes 20GB
New-VHD -Path c:\temp\vms\linux-1\linux-1.vhdx -SizeBytes 20GB
New-VHD -Path c:\temp\vms\linux-2\linux-2.vhdx -SizeBytes 20GB
```

```
New-VM `
-Name "linux-0" `
-Generation 1 `
-MemoryStartupBytes 2048MB `
-SwitchName "virtual-network" `
-VHDPath "c:\temp\vms\linux-0\linux-0.vhdx" `
-Path "c:\temp\vms\linux-0\"

New-VM `
-Name "linux-1" `
-Generation 1 `
-MemoryStartupBytes 2048MB `
-SwitchName "virtual-network" `
-VHDPath "c:\temp\vms\linux-1\linux-1.vhdx" `
-Path "c:\temp\vms\linux-1\"

New-VM `
-Name "linux-2" `
-Generation 1 `
-MemoryStartupBytes 2048MB `
-SwitchName "virtual-network" `
-VHDPath "c:\temp\vms\linux-2\linux-2.vhdx" `
-Path "c:\temp\vms\linux-2\"

```

Setup a DVD drive that holds the `iso` file for Ubuntu Server

```
Set-VMDvdDrive -VMName "linux-0" -ControllerNumber 1 -Path "C:\temp\ubuntu-22.04.3-live-server-amd64.iso"
Set-VMDvdDrive -VMName "linux-1" -ControllerNumber 1 -Path "C:\temp\ubuntu-22.04.3-live-server-amd64.iso"
Set-VMDvdDrive -VMName "linux-2" -ControllerNumber 1 -Path "C:\temp\ubuntu-22.04.3-live-server-amd64.iso"
```

Start our VM's

```
Start-VM -Name "linux-0"
Start-VM -Name "linux-1"
Start-VM -Name "linux-2"
```

Now we can open up Hyper-v Manager and see our infrastructure. </br>
In this video we'll connect to each server, and run through the initial ubuntu setup. </br>
Once finished, select the option to reboot and once it starts, you will notice an `unmount` error on CD-Rom </br>
This is ok, just shut down the server and start it up again.

# Hyper-V : Setup SSH for our machines

Now in this demo, because I need to copy rancher bootstrap commands to each VM, it would be easier to do so
using SSH. So let's connect to each VM in Hyper-V and setup SSH. </br>
This is because `copy+paste` does not work without `Enhanced Session` mode in Ubuntu Server. </br>




sudo apt update
sudo apt install -y nano net-tools openssh-server
sudo systemctl enable ssh
sudo ufw allow ssh
sudo systemctl start ssh
```

Record the IP address of each VM so we can SSH to it:

```
sudo ifconfig
172.31.40.39
# record eth0
linux-0 IP=192.168.1.106
```
machine password: saad946

In new Powershell windows, let's SSH to our VMs

```
ssh linux-0@192.168.1.106


dnf erase runc -y
or
sudo apt-get remove runc

cat /etc/hostname
rancher.server.com

cat /etc/hosts
serverIP rancher rancher.server.com

curl -sSL https://get.docker.com/ | sh
sudo usermod -aG docker $(whoami)
sudo service docker start

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

sudo apt install firewalld
sudo firewall-cmd --zone=public --add-masquerade --permanent
sudo firewall-cmd --permanent --add-port=2379/tcp
sudo firewall-cmd --permanent --add-port=2380/tcp
sudo firewall-cmd --permanent --add-port=7946/tcp
sudo firewall-cmd --permanent --add-port=8472/tcp
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version

or
https://phoenixnap.com/kb/install-helm

wget https://get.helm.sh/helm-v3.9.3-linux-amd64.tar.gz
tar xvf helm-v3.9.3-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin
rm helm-v3.4.1-linux-amd64.tar.gz
rm -rf linux-amd64
helm version

modprobe br_netfilter

The modprobe br_netfilter command is typically used to load the br_netfilter kernel module in a Linux system. This module is related to the bridge and network filtering functionality in the Linux kernel and is often required for certain network-related tasks, such as setting up Kubernetes or other container orchestration systems.

sudo vi /etc/sysctl.conf

net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=0

sudo sysctl -p

get required subnet (adapter)
ip a

curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.27.5+k3s1" K3S_TOKEN=skillpedia#1 sh -s server --docker --node-ip=172.31.40.39 --advertise-address=172.31.40.39 --cluster-init
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config


https://ranchermanager.docs.rancher.com/pages-for-subheaders/install-upgrade-on-a-kubernetes-cluster

helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
whoami
sudo chown ssm-user:ssm-user ~/.kube/config
kubectl create namespace cattle-system

helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.1/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.13.1 
kubectl get pods --namespace cert-manager

kubectl edit deployment metrics-server -n kube-system
   at args:
      - --kubelet-insecure-tls

helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher.my.org --set bootstrapPassword=admin@ --set rancherImageTag=v2.8-head --set ingress.tls.source=letsEncrypt --set letsEncrypt.email=me@example.org --set letsEncrypt.ingress.class=nginx

by video:

helm install rancher rancher-latest/rancher --namespace cattle-system --set replicas=1 --set hostname=rancher.server.com --set bootstrapPassword=saad946@

https://ozgur-kolukisa.medium.com/installing-rancher-on-k3s-with-helm-charts-309bc7287240

Error: INSTALLATION FAILED: chart requires kubeVersion: < 1.27.0–0 which is incompatible with Kubernetes v1.27.3+k3s1

wget https://github.com/k3s-io/k3s/releases/download/v1.26.6%2Bk3s1/k3s
sudo cp k3s /usr/local/bin/k3s
sudo systemctl restart k3s
kubectl get nodes
helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=rancher.server.com --set bootstrapPassword=saad946@ --set ingress.tls.source=letsEncrypt --set letsEncrypt.email=mail@kolukisa.org --set letsEncrypt.ingress.class=nginx

agent,controller,gitjob,rancher,helm operations job -> running/comopleted

kubectl get po -A
kubectl get secret -n cattle-system
kubectl get secret  bootstrap-secret -oyaml -n  cattle-system
or
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=localhost --set rancherImageTag=v2.8-head --set global.cattle.psp.enabled=false
kubectl port-forward rancher-64b477df78-bv29m 80:80 443:443 -n  cattle-system
kubectl get secret  bootstrap-secret -oyaml -n  cattle-system

## Deploy Sample Workloads

To deploy some sample basic workloads, let's get the `kubeconfig` for our cluster </br>

Set kubeconfig:

```
$ENV:KUBECONFIG="<path-to-kubeconfig>"
```

Deploy 2 pods, and a service:

```
kubectl create ns saad
kubectl -n saad apply -f .\kubernetes\configmaps\configmap.yaml
kubectl -n saad apply -f .\kubernetes\secrets\secret.yaml
kubectl -n saad apply -f .\kubernetes\deployments\deployment.yaml
kubectl -n saad apply -f .\kubernetes\services\service.yaml
```

One caveat is because we are not a cloud provider, Kubernetes does not support our service `type=LoadBalancer`. </br>
For that, we need something like `metallb`. </br>
However - we can `port-forward`

```
kubectl -n saad get svc 
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
example-service   LoadBalancer   10.43.235.240   <pending>     80:31310/TCP   13s

kubectl -n saad port-forward svc/example-service 81:80
```

We can access our example-app on port 81






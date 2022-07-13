# Deploy Kubernetes Cluster on Azure VMs using Kubeadm, CNI, Containerd

## Topology and components:
- 1 VNET, 1 Subnet, 1 Security Group
- 2 VMs: a control plane node and a worker node.
- Control plane / Worker Nodes: Container runtime, Kubeadm, Kubelet, Kubectl
- CNI

## Requirements
- A linux shell (bash, zsh, etc.)
- Azure CLI
- Kubectl
- Ssh client
- Azure subscription

## Steps
1. [Deploy Resource Group, VNET and VMs](#deploy-resource-group-vnet-and-vms)   
2. [Give ssh access to node VMs for setup](#give-ssh-access-to-node-vms-for-setup)  
3. [Deploy control plane node](#deploy-control-plane-node)  
a. [Ssh to control plane node from workstation](#ssh-to-control-plane-node-from-workstation)    
b. [Edit hosts file, swap off, enable overlay and netfilter kernel modules on control plane node](#edit-hosts-file-swap-off-enable-overlay-and-netfilter-kernel-modules-on-control-plane-node)  
c. [Install tools and container runtime on control plane node](#install-tools-and-container-runtime-on-control-plane-node)  
d. [Install kubeadm, kubelet, kubectl on control plane node](#install-kubeadm-kubelet-kubectl-on-control-plane-node)    
e. [Inıtialize control plane](#initialize-control-plane)    
f. [Deploy CNI](#deploy-cni)  
4. [Deploy worker node](#deploy-worker-node)    
a. [Ssh to worker node from workstation](#ssh-to-worker-node-from-workstation)  
b. [Edit hosts file, swap off, enable overlay and netfilter kernel modules on worker node](#edit-hosts-file-swap-off-enable-overlay-and-netfilter-kernel-modules-on-worker-node)    
c. [Install tools and container runtime on worker node](#install-tools-and-container-runtime-on-worker-node)    
d. [Install kubeadm and kubelet on worker node](#install-kubeadm-and-kubelet-on-worker-node)  
e. [Join cluster](#join-cluster) 


### Deploy Resource Group, VNET and VMs
We can change the varibales below that suits best for our needs

```bash

export Kname='kubeadm'
export Kregion='northeurope'
export Ksnet='kube'
export Kvnetcidr='10.144.0.0/16'
export Ksnetcidr='10.144.0.0/24'
export Kvmsize='Standard_D2ds_v4'
export Kuser='kubeadmin'

az group create -n $Kname -l $Kregion

# Network
az network vnet create \
    --resource-group $Kname \
    --name $Kname \
    --address-prefix $Kvnetcidr \
    --subnet-name $Ksnet \
    --subnet-prefix $Ksnetcidr

az network nsg create \
    --resource-group $Kname \
    --name $Kname

az network nsg rule create \
    --resource-group $Kname \
    --nsg-name $Kname \
    --name $Kname \
    --protocol '*' \
    --priority 1000 \
    --destination-port-ranges '*' \
    --access allow \
    --destination-address-prefixes VirtualNetwork \
    --source-address-prefixes VirtualNetwork

az network vnet subnet update \
    -g $Kname \
    -n $Ksnet \
    --vnet-name $Kname \
    --network-security-group $Kname

# Deploy Kubernetes control plane node VM
az vm create -n kube-controlplane -g $Kname \
--image UbuntuLTS \
--vnet-name $Kname --subnet $Ksnet \
--admin-username $Kuser \
--ssh-key-value @~/.ssh/id_rsa.pub \
--size $Kvmsize \
--nsg $Kname \
--public-ip-sku Standard

# Deploy Kubernetes worker node VM
az vm create -n kube-worker -g $Kname \
--image UbuntuLTS \
--vnet-name $Kname --subnet $Ksnet \
--admin-username $Kuser \
--ssh-key-value @~/.ssh/id_rsa.pub \
--size $Kvmsize \
--nsg $Kname \
--public-ip-sku Standard

```

### Give ssh access to node VMs for setup
We will need to connect control plane and worker plane VMs over ssh to proceed with the next steps. Thus, we need to give ssh access to control plane and worker nodes from our workstation. We have following options:   

Option 1: Enable and request just in time VM access. Currrently available from portal only.
https://docs.microsoft.com/en-us/azure/defender-for-cloud/just-in-time-access-usage?tabs=jit-config-avm%2Cjit-request-avm#tabpanel_1_jit-config-avm  

Option 2: Add ssh access rule manually in NSG for our workstation ip.  

```bash
Kmypubip=$(curl https://ifconfig.co)
az network nsg rule update \
   --nsg-name $Kname \
   --resource-group $Kname \
   --name $Kname \
   --protocol 'TCP' \
   --priority 1100 \
   --destination-port-ranges '22' \
   --source-address-prefix $Kmypubip \
   --access Allow
```

Option 3: Use Azure Bastion service and connect using our browser and the Azure portal.
https://docs.microsoft.com/en-us/azure/bastion/bastion-overview   

### Deploy Control Plane Node

#### Ssh to control plane node from workstation
```bash
export Kcontrolplaneip=$(az vm list-ip-addresses -g $Kname -n kube-controlplane \
--query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" --output tsv)

ssh  $Kuser@$Kcontrolplaneip
```

#### Edit hosts file, swap off, enable overlay and netfilter kernel modules on control plane node

```bash
# Set version variable to be used when installing kubeadm, kubelet and kubectl
export Kver='1.24.1-00'
export Kvers='1.24.1'

sudo -- sh -c "echo $(hostname -i) $(hostname) >> /etc/hosts"
sudo sed -i "/swap/s/^/#/" /etc/fstab
sudo swapoff -a

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

```
#### Install tools and container runtime on control plane node

```bash

sudo apt update
sudo apt upgrade -y
sudo apt -y install ebtables ethtool apt-transport-https ca-certificates curl gnupg containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd

```
#### Install kubeadm, kubelet, kubectl on control plane node
```bash

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt -y install kubelet=$Kver kubeadm=$Kver kubectl=$Kver

sudo apt-mark hold kubelet kubeadm kubectl
kubectl version --client && kubeadm version

```
#### Initialize Control Plane
```bash

# Only on control plane node
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --kubernetes-version=v$Kvers

# Save the part of the output similar to example below to use later on worker nodes.
# sudo kubeadm join 10.144.0.4:6443 --token gmmmmm.bpar12kc0uc66666 --discovery-token-ca-cert-hash sha256:5aaaaaa64898565035065f23be4494825df0afd7abcf2faaaaaaaaaaaaaaaaaa
#
# If we lose or forget to save the above output, then generate new token with the below command on control plane node to use on worker node later.
# sudo kubeadm token create --print-join-command

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```
#### Deploy CNI
```bash

# We use Weaveworks CNI for this example, but we an also use any other CNI that suits our needs.

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

kubectl get nodes

```

### Deploy Worker Node

#### Ssh to worker node from workstation

Be sure to be back in the workstation to connect worker node.

```bash
export Kworkerip=$(az vm list-ip-addresses -g $Kname -n kube-worker \
--query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" --output tsv)

ssh $Kuser@$Kworkerip
```

#### Edit hosts file, swap off, enable overlay and netfilter kernel modules on worker node

```bash
# Set version variable to be used when installing kubeadm and kubelet
export Kver='1.24.1-00'
export Kvers='1.24.1'

sudo -- sh -c "echo $(hostname -i) $(hostname) >> /etc/hosts"
sudo sed -i "/swap/s/^/#/" /etc/fstab
sudo swapoff -a

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

```
#### Install tools and container runtime on worker node

```bash

sudo apt update
sudo apt upgrade -y
sudo apt -y install ebtables ethtool apt-transport-https ca-certificates curl gnupg containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd

```
#### Install kubeadm and kubelet on worker node
```bash

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt -y install kubelet=$Kver kubeadm=$Kver

sudo apt-mark hold kubelet kubeadm
kubeadm version

```
#### Join Cluster
If we saved kubeadm init command output including kubeadm join command with tokens exist from control plane node deployment then skip (a) and follow (b), otherwise first follow (a) on controller plane node to generate new token and come back to worker node for the rest.
```bash

# (a) Generate token on control plane node. Don't forget to add 'sudo' command at the beginning.
# sudo kubeadm token create --print-join-command
#
# (b) Use the output of "kubeadm init" or "kubeadm token create" command that has ran on control plane node as above. 
# Run the output as example below on worker nodes instead of kubeadm init
# sudo kubeadm join 10.144.0.4:6443 --token gmmmmm.bpar12kc0uc66666 --discovery-token-ca-cert-hash sha256:5aaaaaa64898565035065f23be4494825df0afd7abcf2faaaaaaaaaaaaaaaaaa

```

## FAQ

- Q: Should I deploy everything in the repo to make it work?    
A: We don't necessarily need to deploy as is, but need most of the components. 
- Q: Can I deploy applications on top of the cluster? Is it operational?    
A: Yes we can.
- Q: Can I use this in production?  
A: In production it is recommended to add additional security, high availability, backup and disaster recovery setup. Since in this setup we will need to manage all of the components from infrastructure to applications, a managed cluster is strongly recommended.     
- Q: Can I use docker as container runtime?    
A: It depends on our preference, any Kubernetes.io supported CRI conformed runtimes works fine.  
- Q: Can I use a different CNI  
A: Yes 

## Support

No SLA. Continuous development. Use at your own risk. Please read License.

## Contribution

Contributions are welcome. If you see any improvement points please do not hesitate to open an issue or create a Pull Request.


## Copyright

Copyright &copy; 2022.


## License

This document is open source software licensed under the [Apache License 2.0 license](https://opensource.org/licenses/Apache-2.0).

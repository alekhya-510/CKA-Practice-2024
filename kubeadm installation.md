Kubernetes installation using kubeadm:

We need to spinup 3 nodes in AWS : one controlplane node , two worker nodes

1. Controlplane specs : 1 node
   t2.medium
   sg- 
<img width="1561" alt="Screenshot 2024-10-01 at 07 01 19" src="https://github.com/user-attachments/assets/909f03bb-4344-4f89-973b-d4b583e37fae">

2.  Workernodes specs : 2 nodes
    t2.medium
    sg-
<img width="1562" alt="Screenshot 2024-10-01 at 07 02 52" src="https://github.com/user-attachments/assets/183f0f84-84d4-4517-8230-1182512b4450">

# Controlplane node installation :

# Disable Swap using the below commands
  swapoff -a
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Forwarding IPv4 and letting iptables see bridged traffic

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

#sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

#Apply sysctl params without reboot
sudo sysctl --system

#Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay

#Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

#Install container runtime
  curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

#Check that containerd service is up and running
systemctl status containerd

# install runc

curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# install cni plugin

curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz

# Install kubeadm, kubelet and kubectl

   sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.29.6-1.1 kubeadm=1.29.6-1.1 kubectl=1.29.6-1.1 --allow-downgrades --allow-change-held-packages
sudo apt-mark hold kubelet kubeadm kubectl

kubeadm version
kubelet --version
kubectl version --client

#Configure crictl to work with containerd
 
    sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
   
 # initialize control plane :  --apiserver-advertise-address is private ip of vm on cloud
    
    sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --node-name master
    
12. Prepare kubeconfig

    mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

13. Install calico

    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O

kubectl apply -f custom-resources.yaml

Worker node :

all the above steps to be done except kubeadm init to be done on worker node

we need to create directory as below

mkdir -p $HOME/.kube
and from master node copy contents of /etc/kubernetes/admin.conf to worker node $HOME/.kube/config

Then join worker node by running following output on master node 
kubeadm token create --print-join-command
The output need to run on worker node

    

🧭 Kubernetes (kubeadm) 3-Node Cluster on macOS (Apple Silicon) with Vagrant + VirtualBox

This project creates a 3-node Kubernetes cluster (1 control-plane + 2 workers) using
Vagrant, VirtualBox, and the ARM64 Ubuntu 24.04 box cloudicio/ubuntu-server.

✅ Tested on macOS (M1/M2/M3) with VirtualBox 7.x and Vagrant 2.4.x
⚙️ Each VM: 4 GB RAM, 2 vCPUs
🧱 Network: Host-only 192.168.56.0/24

🧩 Prerequisites

Install these tools first:

brew install --cask virtualbox
brew install --cask vagrant


Then verify:

vagrant --version
VBoxManage -v

🚀 Setup

Create the project directory

mkdir kubeadm-3node && cd kubeadm-3node


Create the Vagrantfile

Save this content as Vagrantfile:

# kubeadm 3-node lab for Apple Silicon + VirtualBox + ARM64 Ubuntu 24.04
Vagrant.configure("2") do |config|
  config.vm.box = "cloudicio/ubuntu-server"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 4096
    vb.cpus   = 2
  end

  # ---------- Common base setup ----------
  COMMON_SETUP = <<-'SHELL'
    set -euxo pipefail
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl gpg conntrack
    swapoff -a
    sed -i '/\sswap\s/ s/^/#/' /etc/fstab || true
    cat >/etc/modules-load.d/k8s.conf <<EOF


overlay
br_netfilter
EOF
modprobe overlay || true
modprobe br_netfilter || true
cat >/etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default >/etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sed -i 's#pause:3.8#pause:3.9#' /etc/containerd/config.toml
systemctl enable --now containerd
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key

| gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/
 /"
>/etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable kubelet
SHELL

 # ---------- Control plane ----------
 config.vm.define "cp1" do |cp|
   cp.vm.hostname = "cp1"
   cp.vm.network "private_network", ip: "192.168.56.10"
   cp.vm.provision "shell", inline: COMMON_SETUP

   cp.vm.provision "shell", inline: <<-'SHELL'
     set -euxo pipefail
     echo 'KUBELET_EXTRA_ARGS="--node-ip=192.168.56.10"' >/etc/default/kubelet
     systemctl daemon-reload && systemctl restart kubelet
     systemctl restart containerd
     kubeadm init \
       --apiserver-advertise-address=192.168.56.10 \
       --pod-network-cidr=10.244.0.0/16
     mkdir -p /home/vagrant/.kube
     cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
     chown vagrant:vagrant /home/vagrant/.kube/config
     sed -i 's#https://10\.0\.2\.15:6443#https://192.168.56.10:6443#g' /etc/kubernetes/admin.conf
     sed -i 's#https://10\.0\.2\.15:6443#https://192.168.56.10:6443#g' /home/vagrant/.kube/config
     su - vagrant -c "kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/refs/heads/master/Documentation/kube-flannel.yml"
     su - vagrant -c "kubectl -n kube-flannel patch ds kube-flannel-ds --type='json' \
       -p='[{\"op\":\"add\",\"path\":\"/spec/template/spec/containers/0/args/-\",\"value\":\"--iface=enp0s8\"}]'"
     kubeadm token create --print-join-command >/vagrant/join.sh
     chmod +x /vagrant/join.sh
   SHELL
 end

 # ---------- Worker template ----------
 def worker_node(config, name, ip)
   config.vm.define name do |node|
     node.vm.hostname = name
     node.vm.network "private_network", ip: ip
     node.vm.provision "shell", inline: COMMON_SETUP
     node.vm.provision "shell", inline: <<-SHELL
       set -euxo pipefail
       echo 'KUBELET_EXTRA_ARGS="--node-ip=#{ip}"' >/etc/default/kubelet
       systemctl daemon-reload && systemctl restart kubelet
       for i in $(seq 1 120); do [ -s /vagrant/join.sh ] && break; sleep 2; done
       kubeadm reset -f || true
       bash /vagrant/join.sh
     SHELL
   end
 end

 worker_node(config, "w1", "192.168.56.11")
 worker_node(config, "w2", "192.168.56.12")


end


3. **Start the cluster**

```bash
vagrant up --provider=virtualbox


The first run downloads the Ubuntu box (takes a few minutes).
Once complete, all three nodes boot and join automatically.

🔐 Verify the cluster
vagrant ssh cp1
kubectl get nodes -o wide
kubectl get pods -A


Expected:

NAME   STATUS   ROLES           AGE   VERSION
cp1    Ready    control-plane   ...
w1     Ready    <none>          ...
w2     Ready    <none>          ...

⚙️ Manage from your Mac (optional)
# From the project directory
vagrant ssh cp1 -c "sudo cat /etc/kubernetes/admin.conf" > admin.conf
export KUBECONFIG="$(pwd)/admin.conf"
kubectl get nodes

🧪 Quick test
kubectl create deployment hello --image=nginx --replicas=2
kubectl expose deployment hello --port=80 --type=ClusterIP
kubectl get pods -o wide

🧰 Common commands
Task	Command
Stop cluster	vagrant halt
Start cluster	vagrant up
SSH into nodes	vagrant ssh cp1 / vagrant ssh w1 / vagrant ssh w2
Destroy cluster	vagrant destroy -f
Re-create join token	vagrant ssh cp1 -c "kubeadm token create --print-join-command"
✅ Features

Works on Apple Silicon (ARM64)

Uses VirtualBox (no Parallels required)

Corrects NAT/host-only network mix-ups
(--apiserver-advertise-address=192.168.56.10)

Each node’s kubelet pinned to its host-only IP

Flannel CNI auto-installed and bound to enp0s8

containerd pre-configured for Kubernetes

🧹 Troubleshooting tips

Worker NotReady?
Wait 30-60 s for Flannel to start.
Check with kubectl -n kube-flannel get pods -o wide.

No connectivity from worker to cp1?
Make sure you can ping 192.168.56.10 and nc -vz 192.168.56.10 6443.

Old IP (10.0.2.15) showing up?
Re-init control-plane with the --apiserver-advertise-address=192.168.56.10 flag.

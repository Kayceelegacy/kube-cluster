# kubeadm 3-node lab for Apple Silicon + VirtualBox + ARM64 Ubuntu 24.04
# Box: cloudicio/ubuntu-server (24.04.1) — works on arm64 + VirtualBox

Vagrant.configure("2") do |config|
  config.vm.box = "cloudicio/ubuntu-server"

  # Common provider config
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 4096
    vb.cpus   = 2
  end

  # Shared base provisioning for all nodes
  COMMON_SETUP = <<-'SHELL'
    set -euxo pipefail

    # Basic deps
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl gpg

    # Disable swap for Kubernetes
    swapoff -a
    sed -i '/\sswap\s/ s/^/#/' /etc/fstab || true

    # Kernel modules & sysctl
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

    # Containerd
    apt-get install -y containerd
    mkdir -p /etc/containerd
    if [ ! -f /etc/containerd/config.toml ]; then
      containerd config default >/etc/containerd/config.toml
    fi
    # Use systemd cgroups and align pause image to 3.9
    sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    sed -i 's#registry.k8s.io/pause:3.8#registry.k8s.io/pause:3.9#' /etc/containerd/config.toml || true
    systemctl enable --now containerd

    # Kubernetes packages (v1.30 stable channel; multi-arch)
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
      | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
      >/etc/apt/sources.list.d/kubernetes.list
    apt-get update
    apt-get install -y kubelet kubeadm kubectl conntrack
    apt-mark hold kubelet kubeadm kubectl
    systemctl enable kubelet
  SHELL

  # -------- Control-plane node --------
  config.vm.define "cp1" do |cp|
    cp.vm.hostname = "cp1"
    cp.vm.network "private_network", ip: "192.168.56.10"
    cp.vm.provision "shell", inline: COMMON_SETUP

    cp.vm.provision "shell", inline: <<-'SHELL'
      set -euxo pipefail

      # Ensure kubelet advertises host-only IP
      if ! grep -q -- '--node-ip=192.168.56.10' /etc/default/kubelet 2>/dev/null; then
        echo 'KUBELET_EXTRA_ARGS="--node-ip=192.168.56.10"' >/etc/default/kubelet
        systemctl daemon-reload && systemctl restart kubelet
      fi

      # If already initialized, skip
      if [ ! -f /etc/kubernetes/admin.conf ]; then
        systemctl restart containerd
        kubeadm init \
          --apiserver-advertise-address=192.168.56.10 \
          --pod-network-cidr=10.244.0.0/16
      fi

      # kubectl for vagrant user
      install -d -m 0755 /home/vagrant/.kube
      cp -f /etc/kubernetes/admin.conf /home/vagrant/.kube/config
      chown vagrant:vagrant /home/vagrant/.kube/config

      # Ensure kubeconfig points to 192.168.56.10 (in case kubeadm picked NAT before)
      sed -i 's#https://10\.0\.2\.15:6443#https://192.168.56.10:6443#g' /etc/kubernetes/admin.conf
      sed -i 's#https://10\.0\.2\.15:6443#https://192.168.56.10:6443#g' /home/vagrant/.kube/config
      chown vagrant:vagrant /home/vagrant/.kube/config

      # Install Flannel CNI (works on arm64). Optionally pin iface to host-only NIC.
      # We add --iface=enp0s8 to ensure it binds to the host-only interface.
      su - vagrant -c "kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/refs/heads/master/Documentation/kube-flannel.yml"
      # Patch DaemonSet to add --iface=enp0s8
      su - vagrant -c "kubectl -n kube-flannel patch ds kube-flannel-ds --type='json' \
        -p='[{\"op\":\"add\",\"path\":\"/spec/template/spec/containers/0/args/-\",\"value\":\"--iface=enp0s8\"}]'"

      # Fresh join command for workers (valid token)
      kubeadm token create --print-join-command > /vagrant/join.sh
      chmod +x /vagrant/join.sh
    SHELL
  end

  # Helper to define workers
  def worker_node(config, name, ip)
    config.vm.define name do |node|
      node.vm.hostname = name
      node.vm.network "private_network", ip: ip
      node.vm.provision "shell", inline: COMMON_SETUP

      node.vm.provision "shell", inline: <<-SHELL
        set -euxo pipefail

        # Ensure kubelet advertises host-only IP
        echo 'KUBELET_EXTRA_ARGS="--node-ip=#{ip}"' >/etc/default/kubelet
        systemctl daemon-reload && systemctl restart kubelet

        # Wait for /vagrant/join.sh (created by cp1)
        for i in $(seq 1 120); do
          [ -s /vagrant/join.sh ] && break
          sleep 2
        done

        # Reset and join (idempotent-ish)
        kubeadm reset -f || true
        bash /vagrant/join.sh
      SHELL
    end
  end

  worker_node(config, "w1", "192.168.56.11")
  worker_node(config, "w2", "192.168.56.12")
end

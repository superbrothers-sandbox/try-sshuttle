# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"

  config.vm.box_check_update = false

  config.vm.define "bastion" do |node|
    node.vm.network "private_network", ip: "192.168.33.10"

    node.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
    end
  end

  config.vm.define "k8s" do |node|
    node.vm.network "private_network", ip: "192.168.33.11"

    node.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
    end

    node.vm.provision "shell", inline: <<-'SHELL'
      set -x

      modprobe overlay
      modprobe br_netfilter

      # Setup required sysctl params
      sysctl -w net.bridge.bridge-nf-call-iptables=1
      sysctl -w net.ipv4.ip_forward=1
      sysctl -w net.bridge.bridge-nf-call-ip6tables=1

      # Disable swap
      sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
      swapoff -a
      mount -a

      # Use iptables-legacy
      apt-get update
      apt-get install -y iptables arptables ebtables
      update-alternatives --set iptables /usr/sbin/iptables-legacy
      update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
      update-alternatives --set arptables /usr/sbin/arptables-legacy
      update-alternatives --set ebtables /usr/sbin/ebtables-legacy

      # Install containerd
      apt-get install -y apt-transport-https ca-certificates curl software-properties-common
      curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
      add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
      apt-get update && apt-get install -y containerd.io
      mkdir -p /etc/containerd
      containerd config default > /etc/containerd/config.toml
      systemctl restart containerd

      # Install kubelet, kubeadm, kubectl
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
      echo deb https://apt.kubernetes.io/ kubernetes-xenial main > /etc/apt/sources.list.d/kubernetes.list
      apt-get update
      apt-get install -y kubelet kubeadm kubectl
      apt-mark hold kubelet kubeadm kubectl

      # Init control-plane node
      kubeadm init --apiserver-advertise-address 192.168.33.11

      export KUBECONFIG=/etc/kubernetes/admin.conf

      # Install cni plugin
      kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
    SHELL
  end
end

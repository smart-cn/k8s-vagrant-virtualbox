CPUS_NUMBER = 4
CPU_LIMIT = 90
MEMORY_LIMIT_MASTER = 2048
MEMORY_LIMIT_WORKER = 2048
BOX_IMAGE = "bento/ubuntu-18.04"
KUBEADM_VERSION = "1.21.0-00"

Vagrant.configure("2") do |config|
  config.vm.provision :shell, privileged: true, inline: $install_common_tools

  config.vm.define "k8master-vm" do |master|
    master.vm.provider :virtualbox do |vb|
      vb.name = "k8master-vm"
      vb.memory = MEMORY_LIMIT_MASTER
      vb.cpus = CPUS_NUMBER
      vb.customize ["modifyvm", :id, "--cpuexecutioncap", "#{CPU_LIMIT}"]
      vb.customize ['modifyvm', :id, '--graphicscontroller', 'vmsvga']
    end
    master.vm.box = BOX_IMAGE
    master.vm.hostname = "k8master-vm"
    master.vm.network :private_network, ip: "10.0.0.10"
    master.vm.provision :shell, privileged: false, inline: $provision_master_node
  end

  %w{k8worker1-vm k8worker2-vm}.each_with_index do |name, i|
    config.vm.define name do |node|
      node.vm.provider "virtualbox" do |vb|
        vb.name = "k8worker#{i + 1}-vm"
        vb.memory = MEMORY_LIMIT_WORKER
        vb.cpus = CPUS_NUMBER
        vb.customize ["modifyvm", :id, "--cpuexecutioncap", "#{CPU_LIMIT}"]
        vb.customize ['modifyvm', :id, '--graphicscontroller', 'vmsvga']
      end
      node.vm.box = BOX_IMAGE
      node.vm.hostname = name
      node.vm.network :private_network, ip: "10.0.0.#{i + 11}"
      node.vm.provision :shell, privileged: false, inline: <<-SHELL
sudo /vagrant/join.sh
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.#{i + 11}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
cat /vagrant/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL
    end
  end

  config.vm.provision "shell", inline: $install_multicast
end


$install_common_tools = <<-SCRIPT
# bridged traffic to iptables is enabled for kube-router
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system

set -e
IFNAME=$1
ADDRESS="$(ip -4 addr show $IFNAME | grep "inet" | head -1 |awk '{print $2}' | cut -d/ -f1)"
sed -e "s/^.*${HOSTNAME}.*/${ADDRESS} ${HOSTNAME} ${HOSTNAME}.local/" -i /etc/hosts

# remove ubuntu-bionic entry
sed -e '/^.*ubuntu-bionic.*/d' -i /etc/hosts

# Patch OS
apt-get update && apt-get upgrade -y

# Create local host entries
echo "10.0.0.10 k8master-vm" >> /etc/hosts
echo "10.0.0.11 k8worker1-vm" >> /etc/hosts
echo "10.0.0.12 k8worker2-vm" >> /etc/hosts

# disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Install kubeadm, kubectl and kubelet
export DEBIAN_FRONTEND=noninteractive
apt-get -qq install ebtables ethtool
apt-get -qq update
apt-get -qq install -y apt-transport-https curl
apt-get -qq install -y containerd; mkdir -p /etc/containerd; sudo containerd config default | sudo tee /etc/containerd/config.toml; sudo systemctl restart containerd

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get -qq update
apt-get -qq install -y \
  kubelet=#{KUBEADM_VERSION} \
  kubeadm=#{KUBEADM_VERSION} \
  kubectl=#{KUBEADM_VERSION}
apt-mark hold kubelet kubectl kubeadm

# Set external DNS
sed -i -e 's/#DNS=/DNS=8.8.8.8/' /etc/systemd/resolved.conf
service systemd-resolved restart
SCRIPT

$provision_master_node = <<-SHELL
OUTPUT_FILE=/vagrant/join.sh
KEY_FILE=/vagrant/id_rsa.pub
rm -rf $OUTPUT_FILE
rm -rf $KEY_FILE

# Create key
ssh-keygen -q -t rsa -b 4096 -N '' -f /home/vagrant/.ssh/id_rsa
cat /home/vagrant/.ssh/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys
cat /home/vagrant/.ssh/id_rsa.pub > ${KEY_FILE}

# Start cluster
sudo kubeadm init --apiserver-advertise-address=10.0.0.10 --pod-network-cidr=10.244.0.0/16 | grep -Ei "kubeadm join|discovery-token-ca-cert-hash" > ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl for vagrant and root users
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo mkdir -p /root/.kube
sudo cp -i /etc/kubernetes/admin.conf /root/.kube/config
sudo chown -R root:root /root/.kube

# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.10"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Use our flannel config file so that routing will work properly
kubectl create -f /vagrant/kube-flannel.yml

# Set alias on master for vagrant and root users
echo "alias k=/usr/bin/kubectl" >> $HOME/.bash_profile
sudo echo "alias k=/usr/bin/kubectl" >> /root/.bash_profile

# Install the etcd client
sudo apt install etcd-client

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL

$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL
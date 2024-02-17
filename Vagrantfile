# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"
Vagrant.require_version ">= 2.0.0"

# just a single node is required
NODES = ENV['NODES'] || 1

# Memory & CPUs
MEM = ENV['MEM'] || 4096
CPUS = ENV['CPUS'] || 2

# User Data Mount
#SRCDIR = ENV['SRCDIR'] || "/home/"+ENV['USER']+"/test"
SRCDIR = ENV['SRCDIR'] || "/tmp/vagrant"
DSTDIR = ENV['DSTDIR'] || "/home/vagrant/data"

# Management
GROWPART = ENV['GROWPART'] || "true"

# Minikube Variables
KUBERNETES_VERSION = ENV['KUBERNETES_VERSION'] || "1.29.0"

# Common installation script
$installer = <<SCRIPT
#!/bin/bash

# Update apt and get dependencies
sudo apt-get -y update
sudo apt-get -y upgrade
sudo apt-get install -y zip unzip curl wget socat ebtables git vim conntrack

SCRIPT

$docker = <<SCRIPT
#!/bin/bash

#curl -fsSL https://apt.dockerproject.org/gpg | sudo apt-key add -
#sudo apt-add-repository "deb https://apt.dockerproject.org/repo ubuntu-xenial main"
#sudo apt-get install -y docker-engine=17.03.1~ce-0~ubuntu-xenial

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=arm64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get -y update
sudo apt-get install -y docker-ce
sudo systemctl start docker

sudo groupadd docker
sudo usermod -a -G docker $USER
newgrp docker

SCRIPT

$growpart = <<SCRIPT
#!/bin/bash

if [[ -b /dev/vda3 ]]; then
  sudo growpart /dev/vda 3
  sudo resize2fs /dev/vda3
elif [[ -b /dev/sda3 ]]; then
  sudo growpart /dev/sda 3
  sudo resize2fs /dev/sda3
fi

SCRIPT

$minikubescript = <<SCRIPT
#!/bin/bash

#Install minikube
echo "Downloading Minikube"
curl -q -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-arm64 2>/dev/null
chmod +x minikube
sudo mv minikube /usr/local/bin/

#Install kubectl
echo "Downloading Kubectl"
curl -q -Lo kubectl https://dl.k8s.io/release/v1.29.0/bin/linux/arm64/kubectl 2>/dev/null
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Helm
echo "Downloading Helm"
curl -qL https://get.helm.sh/helm-v3.14.1-linux-arm64.tar.gz 2>/dev/null | tar xzvf -
chmod +x linux-arm64/helm
sudo mv linux-arm64/helm /usr/local/bin/

#Setup minikube
echo "127.0.0.1 minikube minikube." | sudo tee -a /etc/hosts
mkdir -p $HOME/.minikube
mkdir -p $HOME/.kube
touch $HOME/.kube/config

export KUBECONFIG=$HOME/.kube/config

# Permissions
sudo chown -R $USER:$USER $HOME/.kube
sudo chown -R $USER:$USER $HOME/.minikube
sudo chmod go-r $HOME/.kube/config
sudo chmod -R u+wrx $HOME/.minikube

export MINIKUBE_WANTUPDATENOTIFICATION=false
export MINIKUBE_WANTREPORTERRORPROMPT=false
export MINIKUBE_HOME=$HOME
export CHANGE_MINIKUBE_NONE_USER=true
export KUBECONFIG=$HOME/.kube/config

# Disable SWAP since is not supported on a kubernetes cluster
sudo swapoff -a

## Start minikube
sudo -E minikube start -v 4 --vm-driver docker --kubernetes-version v${KUBERNETES_VERSION} --bootstrapper kubeadm --force

## Addons
sudo -E minikube addons  enable ingress

## Configure vagrant clients dir

printf "export MINIKUBE_WANTUPDATENOTIFICATION=false\n" >> /home/vagrant/.bashrc
printf "export MINIKUBE_WANTREPORTERRORPROMPT=false\n" >> /home/vagrant/.bashrc
printf "export MINIKUBE_HOME=/home/vagrant\n" >> /home/vagrant/.bashrc
printf "export CHANGE_MINIKUBE_NONE_USER=true\n" >> /home/vagrant/.bashrc
printf "export KUBECONFIG=/home/vagrant/.kube/config\n" >> /home/vagrant/.bashrc
printf "source <(kubectl completion bash)\n" >> /home/vagrant/.bashrc

# Permissions
sudo chown -R $USER:$USER $HOME/.kube
sudo chown -R $USER:$USER $HOME/.minikube

# Enforce sysctl
sudo sysctl -w vm.max_map_count=262144
sudo echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.d/90-vm_max_map_count.conf

SCRIPT

required_plugins = %w(vagrant-sshfs vagrant-vbguest vagrant-vmware-desktop)

required_plugins.each do |plugin|
  unless Vagrant.has_plugin? plugin
    system "vagrant plugin install #{plugin}"
  end
end

def configureVM(vmCfg, hostname, cpus, mem, srcdir, dstdir)
  vmCfg.vm.box = "spox/ubuntu-arm"

  vmCfg.vm.hostname = hostname
  vmCfg.vm.network "private_network", type: "dhcp", :model_type => "virtio", :autostart => true

  vmCfg.vm.synced_folder '.', '/vagrant', disabled: true
  # sync your laptop's development with this Vagrant VM
  vmCfg.vm.synced_folder srcdir, dstdir, type: "rsync", rsync__exclude: ".git/", create: true

  # VMware Desktop provider
  vmCfg.vm.provider "vmware_desktop" do |provider, override|
    provider.vmx["memsize"] = mem
    provider.vmx["numvcpus"] = cpus

    # Add any other provider-specific configurations here

    override.vm.synced_folder srcdir, dstdir, type: 'rsync', disabled: false, create: true
  end

  # ensure docker is installed # Use our script so we can get a proper support version
  vmCfg.vm.provision "shell", inline: $docker, privileged: false
  # Script to prepare the VM
  vmCfg.vm.provision "shell", inline: $installer, privileged: false
  vmCfg.vm.provision "shell", inline: $growpart, privileged: false if GROWPART == "true"
  vmCfg.vm.provision "shell", inline: $minikubescript, privileged: false, env: {"KUBERNETES_VERSION" => KUBERNETES_VERSION}

  return vmCfg
end

# Entry point of this Vagrantfile
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vbguest.auto_update = false

  1.upto(NODES.to_i) do |i|
    hostname = "minikube-vagrant-%02d" % [i]
    cpus = CPUS
    mem = MEM
    srcdir = SRCDIR
    dstdir = DSTDIR

    config.vm.define hostname do |vmCfg|
      vmCfg = configureVM(vmCfg, hostname, cpus, mem, srcdir, dstdir)
    end
  end
end

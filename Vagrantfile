# encoding: utf-8
# -*- mode: ruby -*-
# vi: set ft=ruby :


Vagrant.configure(2) do |config|
  config.vm.box = "bento/ubuntu-22.04"  # ! 22.10 uses tcp for nfs
  config.vm.box_version = "202303.13.0"

  config.vm.hostname = 'ubuntu-2204'

  config.vm.provider "virtualbox" do |v|
    v.name = 'ubuntu-2204'
    v.memory = 4096
    v.cpus = 2
    v.customize ['modifyvm', :id, '--clipboard', 'bidirectional'] 
  end

  config.vm.network "private_network", ip: "192.168.56.4"
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 443, host: 8443

  # TODO: modify this to use your username
  config.vm.synced_folder '/Users/me/vagrant_share',
    '/opt/shared',
    type: 'nfs_guest'

  # Install package for your VM
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update -y
    apt-get install -y apt-transport-https build-essential ca-certificates curl debian-archive-keyring debian-keyring git gnupg gnupg-agent nfs-common nfs-kernel-server portmap software-properties-common wget
    mkdir -m 0755 -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
    apt-get update -y
    apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    usermod -aG docker vagrant
    wget https://files.lando.dev/installer/lando-x64-stable.deb
    sudo dpkg -i lando-x64-stable.deb
  SHELL
end


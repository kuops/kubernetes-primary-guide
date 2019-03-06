# -*- mode: ruby -*-
# vi: set ft=ruby :

cluster = {
  "kubeadm-master" => { :ip => "10.0.7.101", :mem => 16384 },
  "kubeadm-worker1" => { :ip => "10.0.7.102", :mem => 2048 },
}

Vagrant.configure("2") do |config|
  config.vm.box = 'ubuntu/xenial64'
  config.vm.box_url = "http://mirrors.ustc.edu.cn/ubuntu-cloud-images/xenial/current/xenial-server-cloudimg-amd64-vagrant.box"
  config.vm.box_check_update = false
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.boot_timeout = 600
  config.vm.provider "virtualbox" do |v|
    v.customize [ "modifyvm", :id, "--uartmode1", "disconnected" ]
    v.cpus = 2
  end

  cluster.each_with_index do |(hostname, info), index|
    config.vm.define hostname do |cfg|
      cfg.vm.provider :virtualbox do |vb, override|
        override.vm.network :private_network, ip: "#{info[:ip]}"
        override.vm.hostname = hostname
        vb.name = hostname
        vb.customize ["modifyvm", :id, "--memory", info[:mem], "--hwvirtex", "on"]
      end # end provider
    end # end config
  end # end cluster
  config.vm.provision "shell", inline: <<-SHELL
    #!/bin/bash
    set -ex

    export DEBIAN_FRONTEND=noninteractive

    sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/cloud/cloud.cfg
    sed -i 's/security.ubuntu.com/mirrors.aliyun.com/g' /etc/cloud/cloud.cfg
    systemctl stop cloud-init
    systemctl disable cloud-init

    sudo sed -i 's@archive.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list
    sudo sed -i 's@security.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list

    sudo sed -i '$anameserver 114.114.114.114' /etc/resolvconf/resolv.conf.d/head
    sudo resolvconf -u

    apt-get update 
    apt-get install apt-transport-https ca-certificates curl software-properties-common -y

    # add docker-ce repo
    curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
    
    add-apt-repository \
      "deb https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
      $(lsb_release -cs) \
      stable"

    # apt-cache policy docker-ce
    apt-get update
    apt-get install docker-ce=5:18.09.2~3-0~ubuntu-xenial -y

    cat > /etc/docker/daemon.json <<-EOF
	{
	  "exec-opts": ["native.cgroupdriver=systemd"],
 	  "registry-mirrors": ["https://registry.docker-cn.com"],
	  "log-driver": "json-file",
	  "log-opts": {
	  "max-size": "100m"
	  },
	  "storage-driver": "overlay2"
	}
EOF

    sudo mkdir -p /etc/systemd/system/docker.service.d

    # Restart docker.
    systemctl daemon-reload
    systemctl restart docker

    # add vagrant user to docker group
    usermod -aG docker vagrant

    # install kubeadm
    apt-get update && apt-get install -y apt-transport-https curl
    curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
    echo "deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl 

  SHELL
end

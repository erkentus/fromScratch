# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "bento/centos-7.1"

  config.vm.define "masterslave" do |masterslave|
      masterslave.vm.network "private_network", ip: "192.168.10.10"
      masterslave.vm.hostname = "masterslave"
  end
  config.vm.define "slavedns" do |slavedns|
      slavedns.vm.network "private_network", ip: "192.168.10.11"
      slavedns.vm.hostname = "slavedns"
  end  
  config.vm.define "slave" do |slave|
      slave.vm.network "private_network", ip: "192.168.10.12"
      slave.vm.hostname = "slave"
  end  
  config.vm.define "interfaceslave" do |interfaceslave|
      interfaceslave.vm.network "private_network", ip: "192.168.10.13"
      interfaceslave.vm.hostname = "interfaceslave"
  end
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
  end
end

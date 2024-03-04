# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  
  config.vm.define "backup" do |backup|
    backup.vm.hostname = "backup"
    backup.vm.network "private_network", ip: "192.168.56.160"
    config.vm.disk :disk, size: "2GB", name: "backup_storage"
  end
  
  config.vm.define "client" do |client|
    client.vm.hostname = "client"
    client.vm.network "private_network", ip: "192.168.56.150"
  end
  
end

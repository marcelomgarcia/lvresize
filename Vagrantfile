# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = "centos/7"

  config.ssh.insert_key = false
  config.ssh.private_key_path = ["keys/vagrant",".vagrant/machines/default/virtualbox/private_key"]
  config.vm.provision "file", source: "keys/config",  destination: "~/.ssh/config"
  config.vm.provision "file", source: "keys/vagrant", destination: "~/.ssh/id_rsa"
  config.vm.provision "file", source: "keys/vagrant.pub", destination: "~/.ssh/id_rsa.pub"
  config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/id_rsa"
  config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/id_rsa.pub"
  config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/config"

  config.vm.provider "virtualbox" do |vb|
    # Customize the amount of memory and num. of CPUs on the VM:
    vb.memory = "2048"
    vb.cpus = 2
  end

  # Define the first server. This will be the master of pcs cluster.
  config.vm.define "atta", primary: true do |machine|
    machine.vm.hostname = "atta"
    machine.vm.network "private_network", ip: "192.168.50.10"
    config.vm.provision "ansible" do |ansible|
      ansible.verbose = "v"
    ansible.playbook = "lvresize.yaml"
    end
  end

  # Define the second server. This will be the slave of pcs cluster.
  config.vm.define "flik" do |machine|
    machine.vm.hostname = "flik"
    machine.vm.network "private_network", ip: "192.168.50.11"
  end

end

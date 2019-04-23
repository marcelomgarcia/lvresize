# Resizing a logical volume

Test the resizing of a logical volume with the option to automatically resize the underlying file system: `lvreize -r`.

## Vagrant

Configuration file to create 2 virtual machines. One will be controller from which Ansible will run, and the other where is the logical volume to be expanded.

This intra-Vagrant setup can be [confusing](https://stackoverflow.com/questions/27005400/vagrant-multiple-machines-inter-ssh-key-authentication), but it seems the important thing is to use [Vagrant ssh keys](https://github.com/hashicorp/vagrant/tree/master/keys) to [configure ssh](https://www.vagrantup.com/docs/vagrantfile/ssh_settings.html) in the Vagrant file. The parameter `private_key_path` is an [array](https://ermaker.github.io/blog/2015/11/18/change-insecure-key-to-my-own-key-on-vagrant.html), that Vagrant uses to set the private keys for ssh.

Putting all together, the _ssh_ configuration on the Vagrant file look like this:

    config.vm.provision "file", source: "keys/config",  destination: "~/.ssh/config"
    config.vm.provision "file", source: "keys/vagrant", destination: "~/.ssh/id_rsa"
    config.vm.provision "file", source: "keys/vagrant.pub", destination: "~/.ssh/id_rsa.pub"
    config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/id_rsa"
    config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/id_rsa.pub"
    config.vm.provision "shell", inline: "chmod 600 /home/vagrant/.ssh/config"

The Vagrant keys are copied to the machines, and permissions are set.

## Ansible

Configuring _atta_ to be the Ansible controller to configure _flik_. The Ansible configuration is inside the block where _atta_ is defined, and since Ansible is being installed in one of the guest machines, it's necessary to use the option _ansible_local_:

    config.vm.define "atta", primary: true do |machine|
    (...)
      config.vm.provision :ansible_local do |ansible|
        ansible.verbose = "v"
        ansible.playbook = "lvresize.yaml"
      end
    end

## NTP

Next we [configure NTP](https://www.tecmint.com/install-ntp-server-in-centos/) so the machines have the correct clock. Atta will be the server, and flik the client.

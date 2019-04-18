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
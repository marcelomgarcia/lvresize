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

It can be useful to have the plugin _vagrant-scp_ installed

    PS C:\Users\mgarcia\Documents\Work\lvresize> vagrant plugin install vagrant-scp
    Installing the 'vagrant-scp' plugin. This can take a few minutes...
    Fetching: vagrant-scp-0.5.7.gem (100%)
    Installed the plugin 'vagrant-scp (0.5.7)'!
    PS C:\Users\mgarcia\Documents\Work\lvresize>
    PS C:\Users\mgarcia\Documents\Work\lvresize>
    PS C:\Users\mgarcia\Documents\Work\lvresize> vagrant plugin list
    vagrant-scp (0.5.7, global)
    PS C:\Users\mgarcia\Documents\Work\lvresize>

So we can copy files between the host and guests, and vice-versa

    PS C:\Users\mgarcia\Documents\Work\lvresize\templates> vagrant scp atta:/etc/ntp.conf .
    ntp.conf       100% 2000   539.2KB/s   00:00
    PS C:\Users\mgarcia\Documents\Work\lvresize\templates> ls


        Directory: C:\Users\mgarcia\Documents\Work\lvresize\templates


    Mode                LastWriteTime         Length Name
    ----                -------------         ------ ----
    -a----       24.04.2019     21:50           2000 ntp.conf


    PS C:\Users\mgarcia\Documents\Work\lvresize\templates>

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

Next we [configure NTP](https://www.tecmint.com/install-ntp-server-in-centos/) so the machines have the correct clock. _Atta_ will be the server, and _flik_ the client.

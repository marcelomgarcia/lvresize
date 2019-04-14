# Resizing a logical volume

Test the resizing of a logical volume with the option to automatically resize the underlying file system: `lvreize -r`.

## Vagrant

Configuration file to create 2 virtual machines. One will be controller from which Ansible will run, and the other where is the logical volume to be expanded.

This intra-Vagrant setup can be [confusing](https://stackoverflow.com/questions/27005400/vagrant-multiple-machines-inter-ssh-key-authentication), but it seems the important thing is to use [Vagrang ssh keys](https://github.com/hashicorp/vagrant/tree/master/keys) to [configure ssh](https://www.vagrantup.com/docs/vagrantfile/ssh_settings.html) in the Vagrant file.
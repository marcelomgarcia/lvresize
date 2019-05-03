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

## LVM

Add another disk to the VM _flik_ via VirtualBox interface. Next, identify the new disk, like by looking at `/dev/disk/` directory.

    [vagrant@flik ~]$ sudo ls -l /dev/disk/by-path
    total 0
    lrwxrwxrwx 1 root root  9 Apr 27 14:07 pci-0000:00:01.1-ata-1.0 -> ../../sda
    lrwxrwxrwx 1 root root 10 Apr 27 14:07 pci-0000:00:01.1-ata-1.0-part1 -> ../../sda1
    lrwxrwxrwx 1 root root  9 Apr 27 14:07 pci-0000:00:01.1-ata-1.1 -> ../../sdb
    [vagrant@flik ~]$

Create a new partition with `parted`, and format it. Although formating the partition can be useless since the `pvcreate` will remove the label.

    [vagrant@flik ~]$ sudo parted /dev/sdb
    (parted) print
    Model: ATA VBOX HARDDISK (scsi)
    Disk /dev/sdb: 100%
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags:

    Number  Start  End  Size  File system  Name  Flags

    (parted) mkpart primary ext3 0% 100%
    (parted) print
    Model: ATA VBOX HARDDISK (scsi)
    Disk /dev/sdb: 100%
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags:

    Number  Start  End   Size  File system  Name     Flags
    1      0.01%  100%  100%               primary

    (parted) quit
    Information: You may need to update /etc/fstab.

    [vagrant@flik ~]$ sudo mkfs.ext3 /dev/sdb1
    mke2fs 1.42.9 (28-Dec-2013)

Then we initialize the physical volume

    [vagrant@flik ~]$ sudo pvcreate /dev/sdb1
    WARNING: ext3 signature detected on /dev/sdb1 at offset 1080. Wipe it? [y/n]: y
    Wiping ext3 signature on /dev/sdb1.
    Physical volume "/dev/sdb1" successfully created.
    [vagrant@flik ~]$

Checking the new disk with `lvmdiskscan` command

    [vagrant@flik ~]$ sudo lvmdiskscan
    /dev/sda1 [     <40.00 GiB]
    /dev/sdb1 [     <18.00 GiB] LVM physical volume
    0 disks
    1 partition
    0 LVM physical volume whole disks
    1 LVM physical volume
    [vagrant@flik ~]$
    [vagrant@flik ~]$
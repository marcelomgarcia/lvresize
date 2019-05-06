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

Create a new partition with `parted`:

    [root@flik ~]# parted /dev/sdb
    (...)
    (parted) print
    Error: /dev/sdb: unrecognised disk label
    Model: ATA VBOX HARDDISK (scsi)
    Disk /dev/sdb: 5369MB
    Sector size (logical/physical): 512B/512B
    Partition Table: unknown
    (...)
    (parted) unit %
    (parted) mklabel gpt
    (parted) mkpart primary ext3 0% 100%
    (parted) print
    Model: ATA VBOX HARDDISK (scsi)
    Disk /dev/sdb: 100%
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags:

    Number  Start  End   Size  File system  Name     Flags
    1      0.02%  100%  100%               primary

    (parted) quit
    Information: You may need to update /etc/fstab.

    [root@flik ~]#

Then we initialize the physical volume

    [root@flik ~]# pvcreate /dev/sdb1
    Physical volume "/dev/sdb1" successfully created.
    [root@flik ~]#

Checking the new disk with `lvmdiskscan` command

    [root@flik ~]# lvmdiskscan
    /dev/sda1 [     <40.00 GiB]
    /dev/sdb1 [      <5.00 GiB] LVM physical volume
    0 disks
    1 partition
    0 LVM physical volume whole disks
    1 LVM physical volume
    [root@flik ~]#

Creating the _Volume_Group_ (VG)

    [root@flik ~]# vgcreate vg1 /dev/sdb1
    Volume group "vg1" successfully created
    [root@flik ~]#

Checking the details of the VG

    [root@flik ~]#
    [root@flik ~]# vgdisplay vg1
    --- Volume group ---
    VG Name               vg1
    System ID
    Format                lvm2
    Metadata Areas        1
    Metadata Sequence No  1
    VG Access             read/write
    VG Status             resizable
    MAX LV                0
    Cur LV                0
    Open LV               0
    Max PV                0
    Cur PV                1
    Act PV                1
    VG Size               <5.00 GiB
    PE Size               4.00 MiB
    Total PE              1279
    Alloc PE / Size       0 / 0
    Free  PE / Size       1279 / <5.00 GiB
    VG UUID               75G1fm-u7NZ-Cx2R-Ef2T-2rzr-gkym-Z2Ep5x

    [root@flik ~]#

Creating the logical volume, and then checking the details of it.

    [root@flik ~]# lvcreate -L 2GB --name lvtest vg1
    Logical volume "lvtest" created.
    [root@flik ~]#
    [root@flik ~]# lvdisplay
    --- Logical volume ---
    LV Path                /dev/vg1/lvtest
    LV Name                lvtest
    VG Name                vg1
    LV UUID                pHyfd5-gBG7-20bU-L5f6-erIL-PsYU-4Eq0tw
    LV Write Access        read/write
    LV Creation host, time flik, 2019-05-05 12:05:04 +0000
    LV Status              available
    # open                 0
    LV Size                2.00 GiB
    Current LE             512
    Segments               1
    Allocation             inherit
    Read ahead sectors     auto
    - currently set to     8192
    Block device           253:0

    [root@flik ~]#

Then we create the file system on the logical volume

    [root@flik ~]# mkfs.ext4 /dev/mapper/vg1-lvtest
    mke2fs 1.42.9 (28-Dec-2013)
    Filesystem label=
    OS type: Linux
    (...)

Next we mount the new file system

    [root@flik ~]# mount /dev/mapper/vg1-lvtest /scratch
    [root@flik ~]# ls /scratch
    lost+found
    [root@flik ~]#
    [root@flik ~]# df -h /scratch
    Filesystem              Size  Used Avail Use% Mounted on
    /dev/mapper/vg1-lvtest  2.0G  6.0M  1.8G   1% /scratch
    [root@flik ~]#

Finally we try to increase the logical volume and the file system using the option `-r` of the `lvresize` command

    [root@flik ~]# umount /scratch
    [root@flik ~]# lvresize  -L +2G -r /dev/vg1/lvtest
    fsck from util-linux 2.23.2
    /dev/mapper/vg1-lvtest: clean, 12/131072 files, 26157/524288 blocks
    Size of logical volume vg1/lvtest changed from 2.00 GiB (512 extents) to 4.00 GiB (1024 extents).
    Logical volume vg1/lvtest successfully resized.
    resize2fs 1.42.9 (28-Dec-2013)
    Resizing the filesystem on /dev/mapper/vg1-lvtest to 1048576 (4k) blocks.
    The filesystem on /dev/mapper/vg1-lvtest is now 1048576 blocks long.

    [root@flik ~]#
    [root@flik ~]#
    [root@flik ~]# mount /dev/mapper/vg1-lvtest /scratch
    [root@flik ~]# df -h /scratch/
    Filesystem              Size  Used Avail Use% Mounted on
    /dev/mapper/vg1-lvtest  3.9G  8.0M  3.7G   1% /scratch
    [root@flik ~]#
    root@flik ~]#  ls /scratch/
    ello_world.txt  lost+found
    root@flik ~]# cat /scratch/hello_world.txt
    ello world of LVM.
    root@flik ~]#
    root@flik ~]#
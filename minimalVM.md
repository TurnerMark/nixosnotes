## 00 Setting up the macine environment in windows 10.
## from Starting life in a VM # July 30th 2023
This workshop takes you through starting life with nixos in a VM on windows 10/11.

Firstly you need to setup your windows installation to do linux stuff, the beginning is to install and enable wsl - windows subsystem for linux,
a good intro tutorial is here on microsoft website :- https://learn.microsoft.com/en-us/windows/wsl/

Then you need the latest --virtualBox-- https://www.virtualbox.org/
currently version 7.0 released July 18th 2023

Then go to the nixos website here :- https://nixos.org/download.html

and get the latest version of the minimal ISO for your platform. For me that is 

https://channels.nixos.org/nixos-23.05/latest-nixos-minimal-x86_64-linux.iso

Assuming you have everything installed now, start virtualbox from the start menu, 
so
       Click the blue sunflower icon for a new VM and give it a name, I suggest "nixos" which is what I will use here.
       Create a local folder for your VMs and set this in election box,   *** Currently not working ***
       Choose your iso image from your downloads folder,
       Select type linux,
          at the bottom of the list of varients choose other 64 bit
       then click next
       in hardware I give it 8Gbytes RAM, and 2 CPUs
       click next
       then add a hard disk - I add 30 to 50Gbytes or so for me this is in M.2 space.

now close virtualbox ansd open powershell to enter the next command to enable nested virtualisation on AMD CPU
assuming you have called your VM nixos, to enable "VT-x/AMD-V" acceleration type the command :-
{
       VBoxManage modifyvm nixos --nested-hw-virt on
}
restart VirtualBox and it should now be ticked.

finally add some shared workspace to your VM - I have a G:\ drive which I map using the shared folders option for my VM

after creating the drive and starting the VM I get the nixos install menu, and pick the first option.

Assuming nothing bad happens on the way, you should get the nixos@nixos bash prompt.

next change the root password using 
{
       sudo passwd root
}
and identify the LAN connection using
{
       ifconfig -a
}
you can now ssh into your new installer VM which will allow you to copy and paste scripts in to it.

In order to login as root without any password, we need to set up our keys, and according to https://nixos.wiki/wiki/SSH_public_key_authentication we start here

In your host machine shell, in your own userspace, make a directory called serverkeys, and enter it.
{
       ssh-keygen -b 1024 -f serverkeys/id_rsa
}
this will require passphrase or blank keyboard input, and will write the files 

serverkeys/id_rsa
serverkeys/id_rsa.pub

then copy your own public key into serverkeys as authorized_keys file
{
       cat ~/.ssh id_rsa.pub >> serverkeys/authorized_keys
}
once you have your keys set up on your host system you can login to the VM using ssh :- 
{
       ssh root@192.168.xx.xxx
}
## Formatting the Virtual Drive

Firstly familiarise yourself with switching between PC and virtual machine using the RHS CTRL key on your keypad -> its the lower right hand key on the standard keyboard.

Now that we have a booted VM, with a virtual drive we can set the partition table.
As I am using an M.2, or SSD  I dont want a swap space, so make a standard msdos partition table.

A note on the manual for GNU Parted states that cheap flash drives should be 4K block aligned, so we insert could insert a special partition to do this, 
and as we also need a boot partition I will attempt to combine forced alignment and boot partition at 32Mb - so we had crashes and gone back to standard partition.

so using parted the script is
{
  parted -s /dev/sda -- mklabel msdos
  parted /dev/sda -- mkpart primary ext4 1 -1s
  parted /dev/sda -- set 1 boot on
  mkfs.ext4 -j -L nixos /dev/sda1
}
so can we use this alignment partition for boot ? we will see .

verify the setup by running fdisk and using the p command to print your partition table.
  you should have sda1 as ext4 label nixos with all space in your Virtual drive.
{
       mount LABEL=nixos /mnt
}
so now we have the target system filesystems in place we can use the script to generate the config file
{
       nixos-generate-config --root /mnt
}
Two files have now been written into /mnt/etc/nixos, these are configuration.nix and hardware-configuration.nix

next we need to edit our configuration.nix to specify our installation.

to do this we edit the configuration file /mnt/etc/nixos/configuration.nix

we need to set 
boot.loader.grub.device = "/dev/sda";
networking.hostName = "nixos";
time.timeZone = "Europe/London";

and Enable the ssh daemon
services.openssh.enable = true;
networking.firewall.enable = "false";

at this stage we should be able to boot into our new VM, but ssh login does not yet work, so we need to fix that up.

In order to configure the installer to make the system image we want, we need to put a good working configuration.nix into the target filesystem.
I have a minimum working configuration.nix stored on my shared drive, together with all the host keys for the server, so we will prep our installation for that.

At the VM make some directories as below
{
       mkdir /mnt/root
       mkdir /mnt/root/.ssh
       mkdir /root/.ssh
}
on your host machine go to your .ssh directory and copy the public key to the VM into the root .ssh folder
with
{
       scp id_rsa.pub root@192.168.xxx.xx:/root/.ssh/authorized_keys  # Put your VM IP address in here.
       scp id_rsa.pub root@192.168.xxx.xx:/mnt/root/.ssh/authorized_keys
}

If we run nixos-install now, upon completion we can ssh directly into our newly installed VM without a password.

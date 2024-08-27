# Ubuntu / Rocky Linux (RHEL) Home Lab Documentation

This repo contains the documentation for a homelab project in which I:

- created two linux virutal machines and networked them together using the built-in VLAN capabilites of VirtualBox
- installed and configured ssh in order to remotely access the Rocky machine from my Ubuntu machine
- installed and configured zfs and nfs to improve functionality on both machines' file systems (one machine with zfs installed and one machine accessing the file system via nfs)
- employed Docker and nginx to host a webpage
- installed Suricata to proactively monitor and protect the devices on the network
- created an Ansible playbook to automate basic setup for future machines


<h2>Hypervisor Choice, Installation, Configuration</h2>

For this project, I decided on VirtualBox for my hypervisor. This choice was made almost entirely based on familiarity: I have used VirtualBox a number of times in the past and have no meaningful complaints. Beyond familiarity, I like VirtualBox because it has a robust set of features, is simple to use out of the box, and is relatively painless to install and run on Windows hosts.

The VirtualBox installation process is very straightforward. After downloading the installer (and verifying the checksum!) the launcher walks the user through the installation process. The only default feature I chose to forego was the Python support, as I don’t intend to manipulate the hypervisor or the VMs with Python. Once you’ve confirmed your feature choices, the program is installed and ready for use.

Beyond the initial setup, I created an internal network so that case I can have multiple virtual machines communicate between one-another locally.

<h2>OS Choice, Installation, Configuration</h2>

Keeping with the trend of familiarity, My primary Linux distribution for this project will be Ubuntu 22.04. However, installation instructions will be more-or-less the same as RedHat solutions. For the sake of experimentation and testing I installed Rocky Linux as a secondary environment for this project (and because I already have Ubuntu installed on my hypervisor). Rocky in particular is exciting because it is essentially a successor to CentOS LTS, and perhaps most importantly, can be had completely for free!

The first step was to download the x86-64 ‘minimal’ image for Rocky 9 from the official website and validate the checksum. Once the checksum was verified, I created a new machine inside of VirtualBox and allocated the system resources, then completed the first-time setup inside the OS by creating a new root password and admin account.

_Note: I chose to not allow root ssh logins with a password because I would consider it a safety concern in a production environment. I don't currently inted to log into the account via ssh, so disallowing the feature entirely felt like a good decision_

To create a new sudo account, first create a new user account using the command **sudo adduser username**, then use the command **sudo usermod -aG sudo username** ('wheel' instead of 'sudo' on Ubuntu machines). You can verify it worked by checking the groups for the new account (command: **groups username**).


<h2>VirtualBox VLAN Setup</h2>

I plan to set up a network storage solution, so naturally I need to provide these machines with the ability to communicate over the network. Luckily, VirtualBox makes creating / connecting to a LAN very easy. From the VirtualBox main screen, click on the bullet menu next to the “Tools” heading, then click “Network” from the new menu that appears. Then click the “Create” button on the next screen to create a new virtual LAN. Once that is done, we are ready to connect our machine(s) to the network. For each machine you wish to connect, open their settings inside of VirtualBox, navigate to the “Network” tab, and check the “Enable Network Adapter” box. From the dropdown menu below it titled “Attached to:”, select “NAT Network” (_NOT just "NAT"!_).

To use ssh inside of the machine, use the command **ip --brief addr show** to get the IP address(es) on the machine(s) you want to connect. Then from the other machine, simply use ssh to remote in (**ssh dest.ip.add.res** or **ssh username@dest.ip.add.ress**).


<h2>ZFS Setup and Configuration</h2>

The next task is to improve the machine by replacing the standard filesystem with the more performant and robust ZFS. The performance gains and mirroring capabilities are reason enough to use ZFS wherever possible, but in particular, I could see it being valuable in a future scenario where an organization may want a more robust and reliable filesystem than whatever off-the-shelf drives/solutions they may currently employ. Depending on the hardware, it could even be possible for the org to enjoy considerably faster write speeds and the peace of mind that RAIDZ configurations provide, without spending a dime on new hardware!

To add ZFS functionality to the Ubuntu machine, I began by creating 4 virtual drives and installing the ZFS utilities by running the command **sudo apt install zfsutils-linux**. Once my machine was equipped with the ZFS utilities, I verified that my virtual drives were recognized by the OS by running **sudo fdisk -l**. I then created the pool by using the zpool create command: **sudo zpool create zstorage mirror /dev/sdb /dev/sdc mirror /dev/sdd /dev/sde**. This command results in two vdevs, each with two mirrored disks, in the zpool (named zstorage) with **sudo zfs mount zstorage** (and once again used **sudo zfs mount** to make sure the filesystem mounted successfully).

<h2>NFS Setup and Configuration</h2>

Next, I wanted to allow the other machines on the network to take advantage of ZFS by turning the Unbuntu machine into an NFS server.

<h3>Configuring the NFS Server</h3>
The first step in setting up my Ubuntu machine as an NFS server was to install the required components using the command **sudo apt install nfs-kernel-server**. Once done, launch the NFS server using the command **sudo systemctl start nfs-kernel-server.service**.

With the server package installed and the service running, we can move on to creating the shared folder for use with NFS. I used the command **sudo mkdir /var/nfs/zstorage -p** to create the shared folder in the /var/ directory with default permissions. Because the file system belongs to the host machine, root operations on the client machine will default to nobody:nogroup credentials as a default security measure. As a result, we need to change the directory ownership to match: **sudo chown nobody:nogroup /var/nfs/zstorage**.

With the credentials changed, I can now move into the NFS configuration files to configure sharing rules. Open the config files (/etc/exports *) using nano and configure the pool with a single-line entry: **/var/nfs/zstorage dest.ip.add.ress(rw,sync,no_subtree_check)**.

I designated the shared folder, the client machine IP, and the permissions as follows: _rw_ allows the client machine to read and write to the volume, _sync_ forces NFS to write changes to disk before replying to client requests, and _no_subtree_check_ turns off the subtree checking on the host machine. Subtree checking is when the host machine checks if the requested file is still available in the export tree for each request from the client. Disabling it was suggested to me because it can cause issues when a file is renamed while the client has that file opened. Once these rules are in place, I restarted the NFS server using **sudo systemctl restart nfs-kernel-server**. The system should now be ready to start sharing files via NFS!

<h3>Configuring the NFS Client</h3>

Over on the client machine (running Rocky Linux), the first step is to download the nfs utilities package by using the command **sudo dnf install nfs-utils**.Once the install was complete, it was time to create a folder to act as the local share location. I created a folder in my desired location with default permissions using the command **sudo mkdir -p /nfs/zstorage**. I then checked to see if the command worked using **df -h** (which reports file system disk space, using the “human-readable” -h flag). Lastly, I tested NFS access by created a file in the share from the client machine (**sudo touch /nfs/zstorage/client-test.txt**).




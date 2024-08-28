# Ubuntu / Rocky Linux (RHEL) Home Lab Documentation

This repo contains the documentation for a homelab project in which I:

- created two linux virutal machines and networked them together using the built-in VLAN capabilites of VirtualBox
- installed and configured ssh in order to remotely access the Rocky machine from my Ubuntu machine
- installed and configured zfs and nfs to improve functionality on both machines' file systems (one machine with zfs installed and one machine accessing the file system via nfs)
- employed Docker and nginx to host a webpage
- created an Ansible playbook to automate basic setup for future machines

The goal of this documentation is to catalogue the process of setting up a simple lab (with a few more considerations given to security than I typically see in write-ups like this) in case myself or anyone else wanted to quickly set up a similar environment. Because the actual uses for applications like Docker and Ansible will vary greatly from person to person, this document simply provides a "scaffolding" for setting up the lab environment.


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

<h2>NFS Configuraion</h2>

Next, I wanted to allow the other machines on the network to take advantage of ZFS by turning the Unbuntu machine into an NFS server.


<h3>Configuring the NFS Server</h3>

The first step in setting up my Ubuntu machine as an NFS server was to install the required components using the command **sudo apt install nfs-kernel-server**. Once done, launch the NFS server using the command **sudo systemctl start nfs-kernel-server.service**.

With the server package installed and the service running, we can move on to creating the shared folder for use with NFS. I used the command **sudo mkdir /var/nfs/zstorage -p** to create the shared folder in the /var/ directory with default permissions. Because the file system belongs to the host machine, root operations on the client machine will default to nobody:nogroup credentials as a default security measure. As a result, we need to change the directory ownership to match: **sudo chown nobody:nogroup /var/nfs/zstorage**.

With the credentials changed, I can now move into the NFS configuration files to configure sharing rules. Open the config files (/etc/exports *) using nano and configure the pool with a single-line entry: **/var/nfs/zstorage dest.ip.add.ress(rw,sync,no_subtree_check)**.

I designated the shared folder, the client machine IP, and the permissions as follows: _rw_ allows the client machine to read and write to the volume, _sync_ forces NFS to write changes to disk before replying to client requests, and _no_subtree_check_ turns off the subtree checking on the host machine. Subtree checking is when the host machine checks if the requested file is still available in the export tree for each request from the client. Disabling it was suggested to me because it can cause issues when a file is renamed while the client has that file opened. Once these rules are in place, I restarted the NFS server using **sudo systemctl restart nfs-kernel-server**. The system should now be ready to start sharing files via NFS!


<h3>Configuring the NFS Client</h3>

Over on the client machine (running Rocky Linux), the first step is to download the nfs utilities package by using the command **sudo dnf install nfs-utils**.Once the install was complete, it was time to create a folder to act as the local share location. I created a folder in my desired location with default permissions using the command **sudo mkdir -p /nfs/zstorage**. I then checked to see if the command worked using **df -h** (which reports file system disk space, using the “human-readable” -h flag). Lastly, I tested NFS access by created a file in the share from the client machine (**sudo touch /nfs/zstorage/client-test.txt**).

<h2>Docker + Nginx Installation and Setup</h2>

With Docker installed, hosting container-based services is made relatively simple. Using nginx to host a static webpage is just one small example of what is possible with just Docker and a little bit of know-how!

Docker can be installed the same way as most other software packages on Ubuntu: **sudo apt-get install docker.io**. You can verify the download worked by checking the installed Docker version using **docker --version**.

Next, I want to allow my non-root profiles to manage Docker. This is accomplished by adding the account(s) to the Docker group: **sudo usermod -aG docker username**. Once added to the group, the user will be able to manage Docker without root privileges next time they log in.

Now that the appropriate permissions are set, we can import an nginx image directly from Docker by using the command **docker pull nginx**. This command automatically pulls down the latest official Docker-supported image onto the machine. We can verify that the download was successful by calling **docker images** to list all of the currently installed images on the machine, where an entry called "nginx" should now be present.

Equipped with the newly-installed nginx image, we can set up a container to serve the image using the command **docker run –name my-nginx -p 8080:80 -d nginx**. This command will spin up a docker container called “my-nginx” using the nginx image from Docker. The -p flag lets us bind ports from the container to our host machine (in this case port 80 on the container to port 8080 on the host). Lastly, the -d flag lets us start the container in detached (“background”) mode. Because we started the container in detached mode, the only response we will get on the terminal is a long string of characters (this is the ID of the container you just made!). We can use the command **docker ps** to list all of the currently running containers. You should see your “my-nginx” container here.

Finally, let's access the container from a browser to confirm it is running the way we expect it to. Navigate to http//localhost:8080 in a web browser on the host machine. If successful, you will see the default "Welcome to nginx" page!


<h2>Bonus: System Setup Automation with Ansible</h2>

Setting up new machines can be an interesting and enjoyable process (if you're a nerd like me!), but setting up a fleet of machines like you might in an enterprise environment is simply too time intensive for most organizations. To solve this problem, there are a number of powerful automation tools available that make system setup and maintenance a breeze. Ansible is one of those tools. It strikes a good balance of being user-friendly while offering a robust set of tools, giving administrators the ability to spin up large, dynamic systems with just a few lines in a yaml file called "playbooks". Ansible playbooks are made up of individual tasks or "plays" that will be executed automatically upon running the playbook. I'm going to automate a few of the setup tasks I typically do manually when setting up environments similar to the one described in this document.

<h3>Creating a Sudo User</h3>

We always like to keep use of the root account to an absolute minimum for both convenience and security purposes. So naturally, the first thing we will need to do with a fresh machine is to create a new sudo user account. On your control machine, create a new ansible playbook file using your preferred editor (I use nano): **nano playbook.yml**. Inside that file, start by adding some configuration info:

```
- hosts: all
  become: true
  vars:
    admin_username: admin
```

The “hosts” category tells Ansible which machines to execute the playbook on, “become” says whether or not commands should be done with root privileges, and “vars” allows us to create variables for any pertinent data for our commands in this playbook. Right now we’re creating a variable called ‘admin_username’ that stores the string ‘admin’.

As mentioned earlier, our interest in using sudo is not only a method to improve security, but also an attempt at improving workflow and convenience. Instead of having to log out of your regular user account and into the root account repeatedly, we can make use of the sudo command. To further decrease the friction of issuing privileged commands, we are setting our rules to not require a password every time sudo is called. This can be accomplished with a simple change in the sudo file. Let's write a task to disable sudo passwords inside our playbook:

```
- name: setup sudo user (no password)
  lineinfile:
    path: /etc/sudoers
    state: present
    regexp: '^%sudo'
    line: '%sudo ALL=(ALL) NOPASSWD: ALL'
    validate: '/usr/sbin/visudo -cf %s'
```

In the above snippet, we are using the lineinfile function provided by Ansible to replace a line (targeted using the regex statement ‘^%sudo’) inside the file located at ‘/etc/sudoers’. The line in the file is replaced with the string stored in ‘line’ inside our play above. As suggested by Ansible, we then use ‘visudo’ to validate our changes to hopefully prevent any issues. Now that that is taken care of, we can write the task to actually add a new sudo user:

```
- name: create a new user ‘admin’ with sudo privileges
  user:
    name: "{{ admin_username }}"
    state: present
    groups: sudo
    append: true
    create_home: true
```

This task checks if we have a sudo user account with name specified in the variable at the top of the file, and if not, creates one (along with a home directory for the new user). If you were to run this playbook as it stands currently, it would disable the sudo password requirement and create a new user named ‘admin’ with sudo privileges automatically. Pretty cool! 

Next, let's continue to improve our security posture and convenience by automating SSH key generation, adding the keys to our authorized_keys file, and disabling root password authentication.

<h3>SSH Setup and Root Authentication</h3>

We’ll start by automating the distribution of the SSH key to the machine. This is doubly advantageous to us as Ansible users, as Ansible assumes that you are using ssh keys for authentication. Typically, you would need to manually generate and move ssh keys around using your terminal, but fortunately for us, Ansible makes this very convenient with its built-in ‘authorized_key’ function:

```
- name: create / set authorized key for remote user
  ansible.posix.authorized_key:
    user: "{{ admin_username }}"
    state: present
    key: "{{ lookup('file', lookup('env','HOME') + '/.ssh/id_rsa.pub') }}"
```

We simply provide the ansible.posix.authorized_key module with the variable storing the name of our new account, and tell it to check if any keys are present in the ssh key file. If not, it generates them to fulfill this requirement. As you can tell from the key line above, we use another built-in Ansible function called ‘lookup’ to build out the path to the user’s key. Now that we are performing authentication using SSH keys, lets disable the root password:

```
- name: disable root password authentication
  lineinfile:
    path: /etc/ssh/sshd_config
    state: present
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin prohibit-password'
```

We once again make use of Ansible’s ‘lineinfile’ function to identify a line using regular expressions, and then replace the specified line with a new string defined inside this task. This time, we’re targeting a line inside of ‘/etc/ssh/sshd_config’ that removes access to root privileges by password, making it so that only SSH key authentication will work.

<h3>Updating, Patching, and Preparing the Machine</h3>

Now that we’re automatically creating a new sudo account and configuring it’s ssh key, we are at the point in the setup process where we’d typically remotely login to the machine and start issuing typical setup commands, namely using apt/apt-get to update and upgrade our system, and installing any desired software. This step will vary for everyone, as it will need to include whichever non-default software packages you need for your machine’s purpose. For demonstration purposes, I’ll write a simple task that will start by updating all packages to their latest version, and then specifically download or update a few common system packages used by developers:

```
- name: update all ubuntu packages
  ansible.builtin.apt:
    name: "*"
    state: latest

- name: Update apt and install required system packages
  apt:
    pkg:
      - git
      - nano
      - curl
    state: latest
    update_cache: true
```

As the name implies, the first task in this section simply asks Ansible to update all of our default Ubuntu packages to their latest version via ‘apt’. This is immensely useful and should typically be done on the first launch of any new machine. The second task asks Ansible to download/update and verify the specific packages listed (git, curl, and nano).

<h3>Running the Playbook</h3>

The playbook now has the ability to create a new user, give them sudo permissions (e.g., add them to the sudoers file), disables the password requirement for sudo calls, generates and sets ssh keys for the new user, disables root password authentication, and installs any other packages or applications that we may want installed on a fresh machine – all with just a few lines in the playbook! With just this file, we can command Ansible to automatically execute this setup process on an arbitrary number of similar machines. Now all that is left is to run our playbook for the first time!

First, we’ll need to add our server hostname or IP address to our Ansible hosts file (‘/etc/ansible/hosts’). Under the [server] header, I added a line that reads: “server1 ansible_host: 10.0.2.15”, which is, as you would expect, the IP address of my Ansible server machine. Once that's done, we can run the playbook!

Assuming this is the first time running this playbook, we’ll need to execute the playbook as ‘root’ with the flag ‘-k’ (which allows us to enter the SSH password for our root user, since the remote server will not be setup yet): **ansible-playbook playbook.yml -l server1 -u root -k**. Once the initial call has been made to the playbook, any subsequent calls can be made using the admin account we created: **ansible-playbook playbook.yml -l server1 -u admin**.


# Ubuntu / Rocky Linux (RHEL) Home Lab Documentation

This repo contains the write-up for a homelab project in which I: 

- created two linux virutal machines and networked them together using the built-in VLAN capabilites of VirtualBox
- installed and configured ssh in order to remotely access the Rocky machine from my Ubuntu machine
- installed and configured zfs and nfs to improve functionality on both machines' file systems (one machine with zfs installed and one machine accessing the file system via nfs)
- employed Docker and nginx to host a simple webpage
- installed Suricata to proactively monitor and protect the devices on the network
- created an Ansible playbook to automate basic setup for future machines

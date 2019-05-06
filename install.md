# OKD All-In-One Deployment Helper

## Prerequisites

- Beefy workstation with Hypervisor installed / activated.
- pfSense Applicance to managed the DNS space and routing.
- RHEL or CentOS 7 minimal install, networking enabled.
  - You will need resolvable DNS entries for your host and apps.  For installs on a VM, a pfSense appliance is ideal as it is pretty easy to configure, and it can maintain a steady IP address space for your install.  It also can run BIND DNS for full support of wildcard DNS records.
  - Create a separate block device (VM Disk) for ephemeral docker storage.
  - Set a proper hostname that is resolvable in DNS. (The 'local.lan' DNS suffix is safe to use with pfSense.)

## pfSense configuration notes

Impelementation details will vary.  I installed on Hyper-V:

- details
- more details

## OS Configuration Options

- Create a user to run ansible playbooks.  Add this user to the 'wheel' group to grant sudo rights.  While running install as root really should be fine for a dev environmnet, I am trying to model 'good behavior' here, and thus am attempting to do this 'the right way'.

  > useradd -G wheel ansible

- In addition to allowing password-less ssh, this user also needs to be configured for password-less sudo.  This can be accomplished by using 'visudo' to set the following options for the 'wheel' group:

  > %wheel  ALL=(ALL)       NOPASSWD: ALL

- Verify that the block storage device that you added for docker storage when creating the VM is visible to the VM using 'lsblk', and that it is defined as /dev/sbd.  Configure docker to use the overlay2 driver on this volume by editing /etc/sysconfig/docker-storage-setup as follows:

      STORAGE_DRIVER=overlay2
      DEVS=/dev/sdb
      VG=docker_vg

- You probably will want some kind of persistant storage as well, but it does not appear that you can provision local persistent storage at deploy time.
  - Add an additional virtual disk.  Verify that it can be seen by the OS using 'lsblk'.  It should be surfaced as /dev/sdc.
  - Create a primary partition on the drive using fdisk
  - Prepare this partition for use with NFS and enable NFS services:

        # Format the partition as XFS:
        mkfs.xfs /dev/sdc1

        # Create an 'export' directory and mount sdc1 to it:
        mkdir /export
        mount /dev/sdc1 /export

        # Prepare a directory for the Docker Registry:
        cd /export
        mkdir etcd logging-es logging-es-ops metrics registry
        chmod -R 755 etcd logging-es logging-es-ops metrics registry
        chown nfsnobody:nfsnobody etcd logging-es logging-es-ops metrics registry

        # Enable services required for NFS:
        systemctl enable rpcbind
        systemctl enable nfs-server
        systemctl enable nfs-lock
        systemctl enable nfs-idmap
        systemctl start rpcbind
        systemctl start nfs-server
        systemctl start nfs-lock
        systemctl start nfs-idmap

        #Open the firewall for NFS services:
        firewall-cmd --permanent --zone=public --add-service=nfs
        firewall-cmd --permanent --zone=public --add-service=mountd
        firewall-cmd --permanent --zone=public --add-service=rpc-bind
        firewall-cmd --reload

- This forum suggests that NetworkManager is required and should not be disabled, and also that if /etc/sysconfig/network-scripts/ifcfg-* file for the active interface should have the entry 'NM_CONTROLLED=yes':  
https://github.com/openshift/openshift-ansible/issues/2163  
It is completely unclear to me how name NetworkManager decides 

## Ansible Playbook Configuration

This repository will contain a sample /etc/ansible/hosts file for an all-in-one deployment.  Notes on this file:

### [OSEv3:vars] section

- You will need to override the default memory check, unless you really want to run a 16Gb VM for OKD:

  > openshift_check_min_host_memory_gb = 8

### [nodes] section

Here you will need to indicate that an all-in-one install is being performed.  Any other host options need to be specified on the same line, so you can end up with rather a lot of text with no line break:

    okd1.local.lan `
      openshift_node_group_name='node-config-all-in-one' `
      openshift_docker_options='--selinux-enabled --log-opt max-size=1M `
        --log-opt max-file=3 --log-driver=json-file `
        --authorization-plugin=docker-novolume-plugin'

## Post-installation tasks

User configuration:

- Define users in the htpasswd file:

  > \# htpasswd /etc/origin/master/htpasswd [username]

- Grant cluster admin rights to this user if required:

  > \# oc policy add-role-to-user cluster-admin [username]
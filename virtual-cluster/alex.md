# Creation of Virtual Cluster
*Herein documents the process taken to create a Highly-available VM Cluster*

## Setting up an environment for the creation of the VM cluster
After some research, Google Cloud Platform(GCP) was selected because it provided free credits of $300USD which is plentiful for this test setup. To set up the VM cluster, nested VMs are required. The steps to creating a nested VM image is stated below:
- Create a boot disk from an existing image with an operating system of your choice on GCP
- Use the boot disk to create a custom image with the special license key required for nested virtualization.
- Run the following code on gcloud command-line tool to create the custom nested VM image:   
  ```bash
  gcloud compute images create nested-vm-image \
  --source-disk disk1 --source-disk-zone us-central1-b \
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-opti  ons/global/licenses/enable-vmx" \
  --guest-os-features MULTI_IP_SUBNET
  ```
   Do take note to replace the naming options and zone option based on the setup.   
- The nested VM image will be available on the custom image tap for selection when creating a VM instance.

The full guide to enabling nested virtualization on VM instances in GCP can be found at https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances

## Setting up of Proxmox Virtual Environment
Proxmox VE is a Debian-based server virtualization management platform which allows deployment and management of virtual machines and containers.  

The following describes the steps taken to setup the Proxmox VE on a nested VM in GCP:
- Create a VM instance on GCP using the custom nested VM image(Debian OS) created previously.
- SSH into the VM instances to setup for the Proxmox VE installation.
- Add the Proxmox Repository:   
   ```bash
   echo "deb http://download.proxmox.com/debian/pve stretch pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
   ```
- Download the Proxmox Repository key:   
   ```bash
   wget http://download.proxmox.com/debian/proxmox-ve-release-5.x.gpg -O /etc/apt/trusted.gpg.d/proxmox-ve-release-5.x.gpg
   ```
- Update repository and system: `apt update && apt dist-upgrade`
- Now that all the preparation has been done, proceed to install Proxmox: `apt-get install proxmox-ve`
- Go to GCP --> VPC network --> firewall rules and create a new firewall rule for tcp:8006. This is to allow access to Proxmox VE web interface.
- Done! Access the Proxmox admin web interface by typing https://google-external-IP-address:8006 in the web browser.  
Upon reaching this point, a Debian nested VM hosting Proxmox VE would be created.

## Creating a cluster and adding all nodes
Created 2 additional VM instances based on the previous steps to set up for a 3 nodes VM cluster. Select a node to create the cluster and subsequently adding the rest of the nodes into the cluster. Below documents the steps to creating the "cloudcluster" proxmox cluster:  
- Node 1 is selected to create the cluster. Create the cluster by typing `pvecm create cloudcluster` in Node 1 CLI.
- Edit the etc/pve/corosync file based on the instructions stated in the **Solutions** segment below.
- Add Node 2 & Node 3 into the cluster by typing `pvecm add node-1` in Node 2 & 3 CLI.  
- The cluster "cloudcluster" will be successfully created with all 3 nodes connected.
Even though the setting up of a cluster is simple and only requires a few command, it is only made possible because of the time and effort spend on researching and troubleshooting the problems faced. During the process of setting up a cluster consisting 3 nodes using the Proxmox web interface, a lot of issues arises during the process of adding the nodes into the cluster.  

### Issues found:
1. Netmask of VMs created on GCP are 255.255.255.255, which interferes with the setting up of promox clustering.  
1. GCP does not support multicast.

### Solutions:

1. Due to the default netmask being 255.255.255.255, the corosync file is only able to work on the node which created the file. When the file is sync with another node, error will occur on the corosync service thus preventing the creation of the cluster.
     - To rectify the netmask issue, add `--guest-os-features MULTI_IP_SUBNET` in to the gcloud command for creating nested VM image. (This command has been updated on to the gcloud command for creating the custom nested VM image in the ***"Setting up an environment for the creation of the VM cluster"*** segment)  
1. A simple research online shows that GCP does not support multicast. Therefore configuration needs to be done on the nodes to enable unicast.
     - Ensure that /etc/pve/corosync.conf file is available in the master node. This file is generated when the node creates the cluster.
     - Edit corosync.conf to include this line "transport: udpu" in the totem { } segment.
     - If the file only has read-only access, enter this command `pvecm e 1` to enable editing.
     - Once editing of the corosync file is done, restart the corosync service using `systemctl restart corosync` and also the pve-cluster service using `systemctl restart pve-cluster`.
     - Proceed to add the rest of the nodes to the cluster using `pvecm add <hostname>`. Where hostname is the name or ip of an existing cluster member. ** Do take note to add the nodes in to the cluster one after another, to prevent any errors.

## Setting up of High Availability Cluster
### Prerequisites:
1. Setting up an empty disk on each node using GCP.
1. Creating a second network(eth1) on GCP and add into the nodes. ** Note that this has to be done at the start when the nodes are created on GCP.  

### Setting up of HA cluster using Ceph:
1. Run `apt-get update` follow by `apt-get upgrade` to update the debian packages on all the nodes.
   - If the update prompts an error, proceed to /etc/apt/source.list.d/pve-enterprise.list and change all the urls from https to http.
1. After the update is completed, proceed to install Ceph using `pveceph install` on all existing nodes.
1. Run `pveceph init --network 192.168.0.0/24` on the main node to initialize Ceph on the eth1 network which was setup previously.
1. Using the command `pveceph createmon` on all the nodes will set up the monitoring service on proxmox ve.
1. Create an OSD on each node with the empty disk using `pveceph createosd /dev/sdb`.
1. Create a storage pool using `pveceph createpool poolname -add_storages`. Take note that this command only needs to be run once on any nodes.

## Setting up of Inner VM
### Prerequisites:
1. Download the iso image file on the node, the iso file will be used to create the inner vm.
   - `cd /var/lib/vz/template/iso` to navigate to the directory where the iso file will be stored.
   - `wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-9.5.0-amd64-netinst.iso
   ` to download the iso file to the current location.
1. Create a VM bridge(vmbr0) network to be used by the inner vms.
1. Setup masquerade on vmbr0 network through eth0 network, to allow access to the internet. This can be achieved by adding the follow commands in to the /etc/network/interfaces file under vmbr0 segment:
   ```
   up echo 1 > /proc/sys/net/ipv4/ip_forward
   post-up iptables -t nat -A POSTROUTING -s '100.100.100.0/24' -o eth0 -j MASQUERADE
   post-down iptables -t nat -D POSTROUTING -s '100.100.100.0/24' -o eth0 -j MASQUERADE
   ```  

### Setting up of Inner VM
The setup for the inner vm is done using the proxmox-ve web interface. With the prerequisites completed, the inner vm should be able to do a network installation without any trouble. The network installation is very important as the iso image downloaded contains only the bare minimum requirement to get the OS started. Therefore take note to select the services which is required ie. ssh server, standard system utilities.  

Things to take note:
1. Key in the correct ip network range based on the vmbr0.
1. When prompt for the ip address of the nameserver, key in 169.254.169.254.  

Once the image have been successfully installed, the inner vm will be ready to use. In order to facilitate the use of Ansible which works with ssh protocol, configuration has to be done on the /etc/ssh/sshd_config file:
1. uncomment port 22.
1. uncomment PermitRootLogin and set to yes.
1. uncomment PasswordAuthentication and set to yes.

## Setting up Ansible
1. Install Ansible package on the node(outer vm) by running `apt-get install ansible`.
1. Setup a ssh connection between the outer node and inner vm by running `ssh-keygen` on the node, and subsequently copy the generated key to the inner vm using `ssh-copy-id Inner-Vm-Ip`
1. Add the inner vm ip address in to the /etc/ansible/hosts file.
1. Create a .yml file containing configuration commands in the /etc/ansible directory.
1. Run the .yml file using `ansible-playbook yourfile.yml`

The following shows a simple ansible script for the installation of postfix:
```
---
  - hosts: 100.100.100.3
    user: root

    tasks:
      - name: Set Postfix option hostname
        debconf: name=postfix question="postfix/mailname" value="cloudmando.com" vtype="string"

      - name: Set Postfix option type as internet site
        debconf: name=postfix question="postfix/main_mailer_type" value="'Internet Site'" vtype="string"

      - name: Install Postfix
        apt: package={{ item }} state=installed force=yes update_cache=yes cache_valid_time=3600
        with_items:
          - postfix
          - mailutils
```
There are a lot of ansible scripts for different purposes online and all it takes is some dedicated research to discover the scripts required.
## Setting up VM Cluster on Host-based Hypervisor
While working on this project, a lot of issue arises because of the limitations of GCP. Issues faced are clustering setup on proxmox and inner VMs network issue. Decided to replicate the project on host-based hypervisors to see if the same issues still persist.  
### Limitations of GCP
1. Does not support multicast.
1. Unable to bridge vmbr0 to eth1 port.  

After some research online, two hypervisors software were shortlisted for the purpose of this test setup.
1. Oracle VM Virtualbox
1. VMware Workstation

### Oracle VM Virtualbox
Working on Virtualbox is a breeze, from the setting up of the network connections to the creation of proxmox VMs. Setting up of the cluster for the VMs is also very simple, no configuration on the corosync file is required because there is no such limitations as compared to GCP.  
However, a huge problem occurred after the setting up of the cluster, Proxmox-ve is unable to setup and host inner VMs. An extensive research online returns with result showing that Oracle VM Virtualbox does not support nested VM virtualization and thus proving that all the efforts are in vain...  
### VMware Workstation
Learning from the previous example, a research is conducted on VMware to check if it supports nested VM virtualization. The results showed that VMware supports nested virtualtization and therefore its safe to proceed with the test setup.  
Setting up of the network connections are much more complicated as compared to Virtualbox. Once the network setup is completed, the setting up of the proxmox VMs and cluster is same as Virtualbox.  
However, issue arises yet again during the setting up of inner VMs. When trying to start up the inner VMs, they will be stuck at the BIOS loading page with the CPU usage reaching 100%. Research online shows that this is actually a bug with hosting proxmox on VMware and the solution is to set the inner VM machine type to 2.6. Once that is done, the inner VMs will be able to boot up normally.   
Run the following command in the Proxmox VMs(node):
```
qm set ID -machine pc-i440fx-2.6
```   
After the inner VMs are setup on each nodes, they are able to ping each other across the nodes which is not achievable on GCP.

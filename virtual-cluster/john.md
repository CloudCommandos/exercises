# John's unrefined guide on setting up virtual clusters
Google Cloud Platform (GCP) is used for this activity.
Proxmox VE is used to achieve Virtual Clustering and High Availability.
Proxmox VE (pve) uses Corosync to link up nodes of a cluster.
A 'node' here refers to an outer VM, which is an instance on GCP.

## Setting up the Environment
Before you create and configure an instance, make sure that you have a nested-VM-enabled image on GCP first. 
GCP does not allow an instance to have nested VMs if the instance is not licensed to do so. The license is free anyway. 
Additionally, your image will also need to be configured to support multiple-IP subnets.
Auto link up for Corosync requires the binding address to be part of the node's subnet. 
By default GCP images will produce instances with a netmask of /32, which is a single IP network.
Use the MULTI_IP_SUBNET option to enable multiple-IP subnets.

Create a disk based on Debian 9  on GCP. Then run the following command in Google Cloud Shell.
Replace the values with the actual values used.

```
gcloud compute images create your-nested-vm-enabled-image \
  --source-disk your-disk-name --source-disk-zone asia-southeast1-b \
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx" \
  --guest-os-features MULTI_IP_SUBNET
```

References:
  1. https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances
  1. https://cloud.google.com/vpc/docs/create-use-multiple-interfaces

Also create a new Virtual Private Cloud (VPC) network. 
This is to allow you to create the instances with two network interfaces on separate networks, which is required for HA.


For this activity, we will spin off VMs based on Debian Stretch.
Our initial setup will have 3 outer VMs with 1 inner VM each.
We need two network interfaces. When creating the instance on GCP, set IP forwarding as ON.

| Interface | Network 			|
| ---------	| ----------		|
| eth0 		| 10.148.0.0/20		|
| eth1		| 192.168.0.0/20	|

| Node ID  		| Host Name 	| Internal IP (eth0, eth1)			| Virtual Bridge IP | Inner VM IP		|
| ------------- | ------------- | ------------						| ------------		| ------------		|
| 1  			| instance-1  	| 10.148.10.1/20, 192.168.10.1/20 	| 192.168.21.1/24	| 192.168.21.2/24	|
| 2  			| instance-2 	| 10.148.10.2/20, 192.168.10.2/20 	| 192.168.22.1/24	| 192.168.22.2/24	|
| 3  			| instance-3  	| 10.148.10.3/20, 192.168.10.3/20 	| 192.168.23.1/24	| 192.168.23.2/24	|

By default eth0 will have internet access.
To enable internet access for eth1 and vmbr0, edit /etc/network/interfaces as such:
```
...
auto eth1
iface eth1 inet dhcp
	up echo 1 > /proc/sys/net/ipv4/ip_forward
	post-up iptables -t nat -A POSTROUTING -s '192.168.10.0/24' -o eth0 -j MASQUERADE
	post-down iptables -t nat -D POSTROUTING -s '192.168.10.0/24' -o eth0 -j MASQUERADE
auto vmbr0
iface vmbr0 inet static
	address 192.168.21.1
	netmask 255.255.255.0
	bridge-ports none
	bridge-stp off
	bridge-fd 0
	up echo 1 > /proc/sys/net/ipv4/ip_forward
	post-up iptables -t nat -A POSTROUTING -s '192.168.21.0/24' -o eth0 -j MASQUERADE
	post-down iptables -t nat -D POSTROUTING -s '192.168.21.0/24' -o eth0 -j MASQUERADE
```
On Google Cloud Platform create Routes for the 10.148.0.0/20 network as such:
| Route No.  	| Destination 		| Next Hop		| 
| -------------	| ------------- 	| ------------	| 
| 1  			| 192.168.21.0/24  	| instance-1 	|
| 2  			| 192.168.22.0/24 	| instance-2 	|
| 3 			| 192.168.23.0/24 	| instance-3	|
This allows inner VMs to be able to communicate internally.


Make sure that your public domain is verified with Google if you want to use it.
https://www.google.com/webmasters/verification/home

Follow this guide for the installation of Proxmox VE and creation of virtual bridges. https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_Stretch.

On Google Cloud Platform (GCP) set the firewall rule to allow tcp on port 8006 so that the Proxmox VE's user interface can be accessed. GCP's firewall settings is located under the menu tab NETWORKING > VPC network > Firewall rules.

After setting the firewall rule, you can access the Proxmox VE user interface via https://YourPublicDomain.com:8006.


## Creating Cluster and Adding Nodes 
You can create a cluster using Proxmox. At the Proxmox UI, select 'Datacenter' on the left side bar. 
Then select 'cluster' on the main portview's menu and click on the 'Create Cluster' button.

Alternatively, you can use the command from your first node:
```
pvecm create cluster-name
```

Due to GCP blocking multicast and that Corosync uses multicast by default. We will need to configure Corosync to use unicast instead.
Edit /etc/pve/corosync.conf and add the property ```transport: udpu``` into totem so that the file has something like this:
```
logging {
  debug: off
  to_syslog: yes
}

nodelist {
  node {
    name: instance-1
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 10.148.10.1
  }
}

quorum {
  provider: corosync_votequorum
}

totem {
  cluster_name: nice-cluster
  config_version: 1
  interface {
    bindnetaddr: 10.148.10.1
    ringnumber: 0
  }
  ip_version: ipv4
  secauth: on
  version: 2
  transport: udpu
}
```

Restart pve-cluster service and /etc/pve/corosync.conf will replace /etc/corosync/corosync.conf
```
systemctl restart pve-cluster
```
Then restart corosync service
```
systemctl restart corosync
```

On the other nodes run the following command to add them to the cluster.
If your /etc/hosts is not configured to include your node1's record, use node1's internal IP instead.
```
pvecm add instance-1
```

##Cluster High Availability
Create one hard disk for each of the nodes via GCP UI.
Allocate 50G of disk space for each of them.
Edit each node and attach the hard disks to their respective nodes.
Run the following commands within each node (only run certain commands once at one node as indicated)
```
pveceph install
pveceph init --network 192.168.10.0/24	#Run once at one node, do not use same subnet as vmbr0!
pveceph createmon
pveceph createosd /dev/sdb
pveceph createpool pool_name -add_storages	#Run once at one node
```

## Creating Inner VMs
You need to create a Linux Bridge first before you can create an inner VM using Proxmox. 
At the Proxmox UI, go to System > Network under your outer VM instance and click on the Create button.
You can also create the bridge by editing /etc/network/interfaces and including a set up similar to what we did for vmbr0 above.

### WARNING! Do not set eth0 as your Linux Bridge's port! This will render your outer VM as inaccessible.
Leave the Linux Bridge port as empty.

Make sure that you have the iso file of the image that the inner VM should be based on. 
Download your iso image and place it in the directory /var/lib/vz/template/iso/. 
You will then be able to select the iso file during the creation of the inner VM through Proxmox. 
You can download the iso file with:

```
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-9.5.0-amd64-netinst.iso
```

When installing the debian image, make sure that 'SSH server' and 'standard system utilities' are included in the installation.
SSH server is required so that you can set up the VM straight away with Ansible playbook.

During installation, specify VM IP within vmbr0's network e.g. 192.168.16.21/20

During installation, specify nameserver as 169.254.169.254

After installation, edit /etc/ssh/sshd_config and set PermitRootLogin and PasswordAuthentication as 'yes'.
Then from the VM's node copy ssh key into the VM
```
ssh-copy-id 192.168.16.21
```
After copying over the ssh key, set PermitRootLogin to prohibit-password and PasswordAuthentication to 'no'.


##Ansible Playbook
Install Ansible on node1
```
apt-get update && apt-get install software-properties-common
apt-add-repository ppa:ansible/ansible
apt-get update && apt-get install ansible
```

###Ansible Playbook Roles
Ansible Playbook roles for postfix, dovecot, and nginx installations are available on ansible-galaxy.
Install the roles.
```
ansible-galaxy install debops.postfix
ansible-galaxy install debops.dovecot
ansible-galaxy install debops.nginx
```
An example ansible playbook script (~/.ansible/scripts/postfix.yml) for postfix installation is as follows:
```
---
- hosts: [ 'debops_service_postfix' ]
  remote_user: root
  environment: '{{ inventory__environment | d({})
               | combine(inventory__group_environment | d({}))
               | combine(inventory__host_environment  | d({})) }}'
  roles:
     - role: debops.postfix 
       tags: ['role::postfix']
```

Before running the playbook, the hostname for "debops_service_postfix" should be added into the hosts list of ansible.
```
vim /etc/ansible/hosts

[debops_service_postfix]
192.168.21.2
```

Run the playbook
```
ansible-playbook ~/.ansible/scripts/postfix.yml --ask-pass
```

useful links
https://icicimov.github.io/blog/virtualization/Proxmox-clustering-and-nested-virtualization/
https://pve.proxmox.com/wiki/Proxmox_VE_4.x_Cluster
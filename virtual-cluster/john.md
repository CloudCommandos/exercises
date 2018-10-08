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
```
gcloud compute images create your-nested-vm-enabled-image \
  --source-disk your-disk-name --source-disk-zone asia-southeast1-b \
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx" \
  --guest-os-features MULTI_IP_SUBNET
```
Ref 1: https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances
Ref 2: https://cloud.google.com/vpc/docs/create-use-multiple-interfaces

Also create a new Virtual Private Cloud (VPC) network. 
This is to allow you to create the instances with two network interfaces on separate networks, which is required for HA.


For this activity, we will spin off VMs based on Debian Stretch.
Our initial setup will have 3 outer VMs with 1 inner VM each.

| Node ID  		| Host Name 	| Internal IP 		| Virtual Bridge IP | Inner VM IP		|
| ------------- | ------------- | ------------		| ------------		| ------------		|
| 1  			| instance-1  	| 10.148.10.1/20 	| 10.148.11.1/20	| 10.148.12.1/20	|
| 2  			| instance-2 	| 10.148.10.2/20 	| 10.148.11.2/20	| 10.148.12.2/20	|
| 3  			| instance-3  	| 10.148.10.3/20 	| 10.148.11.3/20	| 10.148.12.3/20	|


Make sure that your public domain is verified with Google if you want to use it.
https://www.google.com/webmasters/verification/home

Follow this guide for the installation of Proxmox VE and creation of virtual bridges. https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_Stretch.

On Google Cloud Platform (GCP) set the firewall rule to allow tcp on port 8006 so that the Proxmox VE's user interface can be accessed. GCP's firewall settings is located under the menu tab NETWORKING > VPC network > Firewall rules.

After setting the firewall rule, you can access the Proxmox VE user interface via https://YourPublicDomain.com:8006.


## Creating Cluster and Adding Nodes 
You can create a cluster using Proxmox. At the Proxmox UI, select 'Datacenter' on the left side bar. 
Then select 'cluster' on the main portview's menu and click on the 'Create Cluster' button.


## Creating Inner VMs
You need to create a Linux Bridge first before you can create an inner VM using Proxmox. At the Proxmox UI, go to System > Network under your outer VM instance and click on the Create button.
### WARNING! Do not set eth0 as your Linux Bridge's port! This will render your outer VM as inaccessible.
Leave the Linux Bridge port as empty first.

Before you create an inner VM, make sure that you have the iso file of the image that the inner VM should be based on. Download your iso image and place it in the directory /var/lib/vz/template/iso/. You will then be able to select the iso file during the creation of the inner VM. The image used for this activity is obtained as such:

```
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-9.5.0-amd64-netinst.iso
```


useful links
https://icicimov.github.io/blog/virtualization/Proxmox-clustering-and-nested-virtualization/
https://pve.proxmox.com/wiki/Proxmox_VE_4.x_Cluster
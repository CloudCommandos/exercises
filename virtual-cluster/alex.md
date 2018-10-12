# ***Creation of Virtual Cluster***
*Herein documents the process taken to create a Highly-available VM Cluster*
***
### *Setting up an environment for the creation of the VM cluster*
>After some research, Google Cloud Platform(GCP) was selected because it provided free credits of $300USD which is plentiful for this test setup. To set up the VM cluster, nested VMs are required. The steps to creating a nested VM image is stated below:
> - Create a boot disk from an existing image with an operating system of your choice on GCP
> - Use the boot disk to create a custom image with the special license key required for nested virtualization.
> - Run the following code on gcloud command-line tool to create the custom nested VM image: `gcloud compute images create nested-vm-image --source-disk disk1 --source-disk-zone us-central1-b --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx" --guest-os-features MULTI_IP_SUBNET`.
Do take note to replace the naming options and zone option based on the setup.   
> - The nested VM image will be available on the custom image tap for selection when creating a VM instance  
>
>The full guide to enabling nested virtualization on VM instances in GCP can be found at https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances
>
***
### *Setting up of Proxmox Virtual Environment*
>Proxmox VE is a Debian-based server virtualization management platform which allows deployment and management of virtual machines and containers.  
>
>The following describes the steps taken to setup the Proxmox VE on a nested VM in GCP:
> - Create a VM instance on GCP using the custom nested VM image(Debian OS) created previously.
> - SSH into the VM instances to setup for the Proxmox VE installation.
> - Add the Proxmox Repository: `echo "deb http://download.proxmox.com/debian/pve stretch pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list`
> - Download the Proxmox Repository key: `wget http://download.proxmox.com/debian/proxmox-ve-release-5.x.gpg -O /etc/apt/trusted.gpg.d/proxmox-ve-release-5.x.gpg`
> - Update repository and system: `apt update && apt dist-upgrade`
> - Now that all the preparation has been done, proceed to install Proxmox: `apt-get install proxmox-ve`
> - Go to GCP --> VPC network --> firewall rules and create a new firewall rule for tcp:8006. This is to allow access to Proxmox VE web interface.
> - Done! Access the Proxmox admin web interface by typing https://google-external-IP-address:8006 in the web browser.  
>
>Upon reaching this point, a Debian nested VM hosting Proxmox VE would be created.
***
### *Creating a cluster and adding all nodes*
> Created 2 additional VM instances based on the previous steps to set up for a 3 nodes VM cluster. Select a node to create the cluster and subsequently adding the rest of the nodes into the cluster. Below documents the steps to creating the "cloudcluster" proxmox cluster:  
> - Node 1 is selected to create the cluster. Create the cluster by typing `pvecm create cloudcluster` in Node 1 CLI.
> - Edit the etc/pve/corosync file based on the instructions stated in the **Solutions** segment below.
> - Add Node 2 & Node 3 into the cluster by typing `pvecm add node-1` in Node 2 & 3 CLI.  
> - The cluster "cloudcluster" will be successfully created with all 3 nodes connected.
>
>Even though the setting up of a cluster is simple and only requires a few command, it is only made possible because of the time and effort spend on researching and troubleshooting the problems faced. During the process of setting up a cluster consisting 3 nodes using the Proxmox web interface, a lot of issues arises during the process of adding the nodes into the cluster.  
>
>**Issues found:**  
> 1) Netmask of VMs created on GCP are 255.255.255.255, which interferes with the setting up of promox clustering.
>  
> 2) GCP does not support multicast.
>
>**Solutions:**  
>
> 1) Due to the default netmask being 255.255.255.255, the corosync file is only able to work on the node which created the file. When the file is sync with another node, error will occur on the corosync service thus preventing the creation of the cluster.
> - To rectify the netmask issue, add `--guest-os-features MULTI_IP_SUBNET` in to the gcloud command for creating nested VM image. (This command has been updated on to the gcloud command for creating the custom nested VM image in the ***"Setting up an environment for the creation of the VM cluster"*** segment)  
>
> 2) A simple research online shows that GCP does not support multicast. Therefore configuration needs to be done on the nodes to enable unicast.
>- Ensure that /etc/pve/corosync.conf file is available in the master node. This file is generated when the node creates the cluster.
>- Edit corosync.conf to include this line "transport: udpu" in the totem { } segment.
>- If the file only has read-only access, enter this command `pvecm e 1` to enable editing.
>- Once editing of the corosync file is done, restart the corosync service using `systemctl restart corosync` and also the pve-cluster service using `systemctl restart pve-cluster`.
>- Proceed to add the rest of the nodes to the cluster using `pvecm add <hostname>`. Where hostname is the name or ip of an existing cluster member. ** Do take note to add the nodes in to the cluster one after another, to prevent any errors.  
***

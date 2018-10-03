# ***Creation of Virtual Cluster***
*Herein documents the process taken to create a Highly-available VM Cluster*
***
### *Setting up an environment for the creation of the VM cluster*
>After some research, Google Cloud Platform(GCP) was selected because it provided free credits of $300USD which is plentiful for this test setup. To set up the VM cluster, nested VMs are required. The steps to creating a nested VM image is stated below:
> - Create a boot disk from an existing image with an operating system of your choice on GCP
> - Use the boot disk to create a custom image with the special license key required for nested virtualization.
> - Run the following code on gcloud command-line tool to create the custom nested VM image: `gcloud compute images create nested-vm-image --source-disk disk1 --source-disk-zone us-central1-b --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"`. Do take note to replace the naming options and zone option based on the setup.   
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
> - Done! Access the Proxmox admin web interface by typing https://youripaddress:8006 in the web browser.  
>
>Upon reaching this point, a Debian nested VM hosting Proxmox VE would be created.
***
### *Creating a cluster and adding all nodes*
> Created 2 more VM instances based on the previous step to serve as nodes for the VM cluster. Tried to set up a cluster using the Proxmox web interface but was faced with a lot of issues during the process of adding the 2nd node to the cluster.

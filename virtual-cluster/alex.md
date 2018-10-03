# **Creation of Virtual Cluster**
*Herein documents the process taken to create a Highly-available VM Cluster*
***
### *Setting up an environment for the creation of the VM cluster*
>After some research, Google Cloud Platform(GCP) was selected because it provided free credits of $300USD which is plentiful for this test setup.  
To set up the VM cluster, nested VMs are required. The steps to creating a nested VM image is stated below;
- Create a boot disk from an existing image with an operating system of your choice on GCP
- Use the boot disk to create a custom image with the special license key required for nested virtualization.
- Run the following code on gcloud command-line tool to create the custom nested VM image: `gcloud compute images create nested-vm-image \
  --source-disk disk1 --source-disk-zone us-central1-b \
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"`. **Do take note to replace the naming options and zone option based on the setup.   
- The nested VM image will be available on the custom image tap for selection when creating a VM instance  
>
>The full guide to enabling nested virtualization on VM instances in GCP can be found at https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances
>
***
### *Setting up of Proxmox Virtual Environment*
>Proxmox VE is a server virtualization management platform which allows deployment and management of virtual machines and containers.

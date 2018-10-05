# John's unrefined guide on setting up virtual clusters
Google Cloud Platform (GCP) is used for this activity.

Before you create and configure an instance, make sure that you have a nested-VM-enabled image on GCP first. GCP does not allow an instance to have nested VMs if the instance is not licensed to do so. The license is free anyway. To create a nested-VM-enabled image, follow this guide https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances. For this activity, we will spin off VMs based on Debian Stretch.

Make sure that your public domain is verified with Google.
https://www.google.com/webmasters/verification/home

Follow this guide for the installation of Proxmox VE and create a virtual bridge, https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_Stretch.

On Google Cloud Platform (GCP) set the firewall rule to allow tcp on port 8006 so that the Proxmox VE's user interface can be accessed. GCP's firewall settings is located under the menu tab NETWORKING > VPC network > Firewall rules.

After setting the firewall rule, you can access the Proxmox VE user interface via https://YourPublicDomain.com:8006.
You need to create a Linux Bridge first before you can create an inner VM using Proxmox. Using the Proxmox UI, go to System > Network under your outer VM instance and click on the Create button.
### WARNING! Do not set eth0 as your Linux Bridge's port! This will render your outer VM as inaccessible.
Leave the Linux Bridge port as empty.

Before you create an inner VM, make sure that you have the iso file of the image that the inner VM should be based on. Download your iso image and place it in the directory /var/lib/vz/template/iso/. You will then be able to select the iso file during the creation of the inner VM. The image used for this activity is obtained as such:

```
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-9.5.0-amd64-netinst.iso
```

useful links
https://icicimov.github.io/blog/virtualization/Proxmox-clustering-and-nested-virtualization/
https://pve.proxmox.com/wiki/Proxmox_VE_4.x_Cluster
Virtual Cluster
===

## Setting up Google Cloud with Nested Virtualisation
---

Reference to https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances

Firstly, create a disk with a iso image. e.g Debien 9 Stretch. Once the disk is created, open up "Activate Cloud Shell" and type in the following:
```
gcloud compute images create nested-vm-image \
  --source-disk yourdiskname --source-disk-zone source-of-yourdisk-zone(e.g asia-southeast1-b) \
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"
```

Once completed, the disk is configured with nested virtualisation.

## Create a instances
---
Go to "Compute Engine" and select "VM instances". Click on "Create instances", at the "Boot disk" option, click "change" and select "Custom images", the disk with nested virtualisation should be available for selection under your project. e.g **nested**-vm-image-debian Complete the rest of the options with the preferred configuration.

## Installing Proxmox
---

Reference to https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_Stretch for installation of Proxmox on Debian Stretch.

ssh into the vm and following the instructions from the above link.

Once the installation is completed, open the link "https://yourpublicipaddress:8006". The browser will not be able to display the Proxmox web interface. This is caused by the firewall rules.

At the VM instances page, select the vm instance and select "view network details". "VPC network" will open up and select "Firewall rules", then select "Create firewall rule". Select "All instances in the network" under the Targets option. Set IP range as "0.0.0.0/0".
Specify Port 8006 for TCP. Once completed, create the firewall rule. Try to login again and the browser will likely show "Your connection is not private". Click "Advance" and then proceed. The browser will now proceed to the Proxmox web interface.  

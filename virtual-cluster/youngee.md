Virtual Cluster
===

## Setting up Google Cloud with Nested Virtualisation

Refer to: https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances for nested virtualisation on Google Cloud.

Firstly, create a disk with a iso image. e.g Debien 9 Stretch.

Command required to enable nested virtualisation: `  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx" `

Continue with the next section to enable multiple subnet on Google Cloud. The final setup command will be shown in the next section.


##Enable Multiple Subnet on Google Cloud

During the setup of proxmox, error will occurred when a node tries to add to a cluster. This issue is caused by the default network configuration on the NIC0 on Google Cloud setting.

Default netmask for the instance of eth0 is `255.255.255.255`. This will caused an issue when a node tries to add to a cluster.

Thus, configuration on the network interface of the instance is required to allow more subnet.

Open up the 'Activate Cloud Shell' on your Google Cloud and enter the following command to enable nested virtualisation and multiple subnet on a disk:
```
gcloud compute images create yourcustomediskname \
     --source-disk yourdiskname \
     --source-disk-zone yourdiskzone \
     --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx" \
     --guest-os-features MULTI_IP_SUBNET
```

A custom image will be created with the command above, with the function enabled for nested virtualisation and multiple subnet.

Refer to: https://cloud.google.com/vpc/docs/create-use-multiple-interfaces#i_am_having_connectivity_issues_when_using_a_netmask_that_is_not_32 for multiple subnet on Google Cloud.

## Create a instances

Go to "Compute Engine" and select "VM instances". Click on "Create instances", at the "Boot disk" option, click "change" and select "Custom images", the disk that was created in the previous section should be available for selection under your project. Complete the rest of the options with the preferred configuration.

Start up the VM instance and verify the nested virtualisation and multiple subnet are enabled.

Use `grep -cw vmx /proc/cpuinfo`, a non-zero response confirms that nested virtualization is enabled.

Use `ifconfig` to check the subnet for eth0. The subnet should be 255.255.240.0 if the multiple subnet is enabled successfully.

Create VPC network, choose a your region and IP range

## Installing Proxmox

Reference to https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_Stretch for installation of Proxmox on Debian Stretch.

ssh into the vm and following the instructions from the above link.

Once the installation is completed, open the link "https://yourpublicipaddress:8006". The browser will not be able to display the Proxmox web interface. This is caused by the firewall rules.

At the VM instances page, select the vm instance and select "view network details". "VPC network" will open up and select "Firewall rules", then select "Create firewall rule". Select "All instances in the network" under the Targets option. Set IP range as "0.0.0.0/0".
Specify Port 8006 for TCP. Once completed, create the firewall rule. Try to login again and the browser will likely show "Your connection is not private". Click "Advance" and then proceed. The browser will now proceed to the Proxmox web interface.  

## Setup Cluster

Once the proxmox is installed, the cluster can be now created on a node. Select a node and use the command `pvecm create mycluster`. In this case, the cluster will reside in node1.

The command above will then create a cluster and using a command `pvecm status` allows you to check the cluster status.

After encountering issued during the setup on the a second node, certain setting was required on the `/etc/pve/corosync.conf` file. Google Cloud does not allow multicast and thus, a second node is unable to add to the cluster. Therefore, unicast is utilise to allow the adding of node to the cluster.

To use Unicast (UDPU), add `transport= udpu` in the totem{} stanza in the `/etc/pve/corosync.conf` file. Restart the service to allow the new configuration to initialise. The corosync service can be restart with `systemctl restart corosync.service`.

Refer to: https://pve.proxmox.com/wiki/Multicast_notes#Use_unicast_.28UDPU.29_instead_of_multicast.2C_if_all_else_fails for Unicast setup on Proxmox.

## Adding Cluster

Once the cluster is setup correctly, ssh into the nodes that needs to be added to the cluster. In this case, node2 and node3 will be added to the cluster.

ssh into node2 and node3 respectively, use the command `pvecm add ip.address.of.node1` e.g pvecm add `10.148.0.2`. This will allow the nodes to add to the cluster created at node1.

To verify if the nodes are added, use the command `pvecm status`

An example of the `pvecm status` output on node1 is as shown:
```
root@node1:~# pvecm status
Quorum information
------------------
Date:             Tue Oct  9 03:12:42 2018
Quorum provider:  corosync_votequorum
Nodes:            3
Node ID:          0x00000001
Ring ID:          1/76
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2
Flags:            Quorate

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 10.148.0.2 (local)
0x00000002          1 10.148.0.3
0x00000003          1 10.148.0.4
```

## Creating inner VM
Firstly, download the required ISO image and store the image at `/var/lib/vz/template/iso`

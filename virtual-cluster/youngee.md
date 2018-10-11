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

  Example of the `/etc/pve/corosync.conf`:
  ```
  logging {
  debug: off
  to_syslog: yes
}
nodelist {
  node {
    name: node1
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 10.148.0.2
  }
  node {
    name: node2
    nodeid: 2
    quorum_votes: 1
    ring0_addr: 10.148.0.3
  }
  node {
    name: node3
    nodeid: 3
    quorum_votes: 1
    ring0_addr: 10.148.0.4
  }
}
quorum {
  provider: corosync_votequorum
}
totem {
  cluster_name: mycluster
   config_version: 3
   interface {
     bindnetaddr: 10.148.0.2
     ringnumber: 0
   }
   ip_version: ipv4
   secauth: on
   transport: udpu
   version: 2
 }
  ```

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

## Setup Ceph for High-Availability
Ceph is software designed to provide highly scalable object, block and file-based storage under a unified system.

Before the installation of Ceph, there are additional task to carry out on Google Cloud. Create a empty disk for each node. The empty disk will be used for the Ceph setup. Additionally, create another VPC network and set a IP range of your choice. Set the region to the same region as the VM instance, in this case is 'asia-southeast1'. Apply the same set of firewall rules as the 'default' VPC network setting.

Before the installation of Ceph, update the Debian packages with `apt-get update && apt-get upgrade`.

If there is a error message shown during the update such as:
```
W: The repository 'https://enterprise.proxmox.com/debian/pve stretch Release' does not have a Release file.
N: Data from such a repository can't be authenticated and is therefore potentially dangerous to use.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: Failed to fetch https://enterprise.proxmox.com/debian/pve/dists/stretch/pve-enterprise/binary-amd64/Packages  40
1  Unauthorized
E: Some index files failed to download. They have been ignored, or old ones used instead.
```

Edit the file, `/etc/apt/source.list.d/pve-enterprise.list` and change the `https` to `http`.

Once the update and upgrade is completed, Ceph can be now install. To install Proxmox Ceph, use the command `pveceph install`.

Once the installation is completed, use this command on **node1** to initialise Ceph `pveceph init --network ipaddressnetworkofeth1/prefix`. e.g `pveceph init --network 192.168.1.0/24` The IP address used is network IP address which was additionally created earlier on the Google Cloud, which should be the nic1 on the VM instances.

Next, use `pveceph createmon` to create the monitor on each node.

Go to the web interface of node1 Proxmox and under the Ceph tab, the status will show HEALTH_OK and under Monitor tab, the 3 nodes should be shown as well. Verify the rest of the 2 nodes, which should also include all the 3 nodes under its Ceph tab. If Ceph its not installed, you will not be able to enter the Ceph tab on the web interface.

At the Configuration tab, the 3 nodes and its respective IP address show be shown. All monitors are up now. Example of the Configuration text:
```
[global]
	 auth client required = cephx
	 auth cluster required = cephx
	 auth service required = cephx
	 cluster network = 192.168.1.0/24
	 fsid = 2d6b97df-9c84-47a3-8d16-9147c7f58ee6
	 keyring = /etc/pve/priv/$cluster.$name.keyring
	 mon allow pool delete = true
	 osd journal size = 5120
	 osd pool default min size = 2
	 osd pool default size = 3
	 public network = 192.168.1.0/24

[osd]
	 keyring = /var/lib/ceph/osd/ceph-$id/keyring

[mon.node1]
	 host = node1
	 mon addr = 192.168.1.2:6789

[mon.node3]
	 host = node3
	 mon addr = 192.168.1.4:6789

[mon.node2]
	 host = node2
	 mon addr = 192.168.1.3:6789
```

Subsequently, create an OSD for each node. This can be done via the web interface under Ceph's OSD tab or using command `pveceph createosd /dev/yourdiskpartition`. Use the empty disk which was created earlier and attached to the VM instances. In this case, the empty disk is mounted as `/dev/sdb`. Thus, using command line will be `pveceph createosd /dev/sdb`. Carry out this command on all 3 nodes.
Example of the output for `pveceph createosd /dev/sdb`:
```
root@node1:~# pveceph createosd /dev/sdb
command '/sbin/zpool list -HPLv' failed: open3: exec of /sbin/zpool list -HPLv failed: No such file or directory at /usr/share/perl5/PVE/Tools.pm line 429.

create OSD on /dev/sdb (bluestore)
Creating new GPT entries.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
Creating new GPT entries.
The operation has completed successfully.
Setting name!
partNum is 0
REALLY setting name!
The operation has completed successfully.
Setting name!
partNum is 1
REALLY setting name!
The operation has completed successfully.
The operation has completed successfully.
meta-data=/dev/sdb1              isize=2048   agcount=4, agsize=6400 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=0, rmapbt=0, reflink=0
data     =                       bsize=4096   blocks=25600, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=1608, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
The operation has completed successfully.
```

Verify the osd for all 3 nodes are created on the web interface under the OSD tab and the 3 nodes should be created and shown.

Thereafter, create a pool for the osd. This can be done via web interface or command `pveceph createpool yourpoolname -add_storages`. The storage pool will now be available on all the 3 nodes.

The shared storage is now available! Next, implement the automatic fail-over with HA on PVE.

## Creating inner VM
Firstly, download the required ISO image and store the image at `/var/lib/vz/template/iso`

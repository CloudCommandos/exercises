# Prototype Molly
We are to provision an operational Kubernetes (k8s) Cluster using 5 servers:

|   Server Name   |   CPU Cores   |   RAM (GiB)   |   Hard Disks   |
|---|---|---|---|
|   pve0   |   6   |   32   |   4 x 1.8TiB   |
|   pve1   |   6   |   32   |   4 x 1.8TiB   |
|   pve2   |   6   |   32   |   4 x 1.8TiB   |
|   pve3   |   6   |   32   |   3 x 1.8TiB   |
|   pve4   |   6   |   32   |   4 x 1.8TiB   |

---
## Base Server Setup
Install Proxmox VE v5.4 on all 5 servers. You can use a flashed a USB thumb drive with the Proxmox image. We set up zfs RAID 1 using 2 hard disks on each server. For our servers, we have to change bootup mode and SATA config to legacy in order for zfs RAID to setup successfully. The Remaining 9 hard disks are used to set up Ceph. Ceph storage are to be used for the provisioning of Debian9.8 virtual machines. CephFS will also be set up to mount volume shares into k8s pods.

---
## Internet Access Provisioning
We will set up pve0 & pve1 with connection to the internet, but only 1 server will be connected to the internet at one time. The traffic in and out of the Proxmox cluster will be monitor by the firewall VM created in the cluster. We decided to go with OPNsense for our firewall because of its intuitive and user-friendly webGUI interface. The setup is as such:

|   Interfaces   |   Firewall 1(pve0)   |   Firewall 2(pve1)   |   CARP/Comment   |
|---|---|---|---|
|   WAN	  |   192.168.24.24/28   |   192.168.24.25/28 |   [Public IP]   |
|   LAN	  |   10.0.1.6/24	  |   10.0.1.7/24   |   10.0.1.5/24   |
|   HA  	|   10.0.1.6/24	  |   10.0.1.7/24   |   Use LAN network because not enough network interface   |

As shown above, both server will be sharing 1 public IP address and only one server will be using the public IP address at one time. This is achieved by using the Common Address Redundancy Protocol(CARP) available on OPNsense.  

### Basic Setup
The following details the steps taken to set up the firewall VMs:  

1. Boot OPNsense iso image

1. Login to OPNsense using the following credentials:
   * ID : installer
   * Password : opnsense   

1. Proceed with the installation setup by choosing default options    

1. Reboot to complete installation  

1. Login using `root` account and the password you set and you should see the following:

   ```
   ----------------------------------------------
   |      Hello, this is OPNsense 19.1          |         @@@@@@@@@@@@@@@
   |                                            |        @@@@         @@@@
   | Website:      https://opnsense.org/        |         @@@\\\   ///@@@
   | Handbook:     https://docs.opnsense.org/   |       ))))))))   ((((((((
   | Forums:       https://forum.opnsense.org/  |         @@@///   \\\@@@
   | Lists:        https://lists.opnsense.org/  |        @@@@         @@@@
   | Code:         https://github.com/opnsense  |         @@@@@@@@@@@@@@@
   ----------------------------------------------
   *** OPN1.localdomain: OPNsense 19.1.4 (amd64/OpenSSL) ***
   LAN (vtnet0)    -> v4: 10.0.1.6/24
   WAN (vtnet1)    -> v4: 192.168.24.24/28

   HTTPS: SHA256 19 A8 83 86 42 4E 75 3C DC 0D AB C0 6F BC 54 91
                 22 ED E8 BB A1 B1 85 E3 A1 80 E5 15 89 9C 99 BC
   SSH:   SHA256 HpmbGXeyvNKd7Hyc4sx+FTR6pL0wytIIVAWDuVrDZMc (ECDSA)
   SSH:   SHA256 p0KjpBlqupzqBXOMot0beFKHKx2EJNgkabxl14CVCZ8 (ED25519)
   SSH:   SHA256 xvRzsU1SjclG9TxiV8D6dsWSUvSx7b5QDn25Y3n2e+U (RSA)

    0) Logout                              7) Ping host
    1) Assign interfaces                   8) Shell
    2) Set interface IP address            9) pfTop
    3) Reset the root password            10) Firewall log
    4) Reset to factory defaults          11) Reload all services
    5) Power off system                   12) Update from console
    6) Reboot system                      13) Restore a backup
   Enter an option:
   ```  
1. Select option 1 to assign the interfaces
   * WAN is for the interface with internet connection
   * LAN is for the interface with internal network

1. Select option 2 to set IP addresses for the interfaces
   * WAN : 192.168.24.24/28 or Public IP address
   * LAN : 10.0.1.6/24

1. Access the web GUI of the firewall using its IP address to complete the installation setup
   * Installation wizard will prompt when you first enter the web GUI
   * Unchecked "Block private networks" option on WAN interface setting
   * Unchecked "Block bogon networks" option on WAN interface setting

1. To enable port forwarding on the firewall:
   * Navigate to Firewall >> Settings >> Advanced
   * Checked "Reflection for port forwards" option
   * Checked "Automatic outbound NAT for Reflection" option  

To use the firewall to route internet to the other VMs, go to the network config of the VMs and set gateway as the firewall. Subsequently, the VMs will be able to access the internet through the firewall.

### CARP Setup using only 1 public IP
The following details the steps taken to create a CARP set up using 1 public IP instead of the conventional 3 public IPs:   
1. Set up both Firewall WAN interface with a dummy IP address

1. Create CARP for WAN interface
   * Using any one of the Firewall(1), navigate to  Firewall >> Virtual IPs >> Settings
   * click on the "Add" button
   * For Mode, select "CARP"
   * For Interface, select "WAN"
   * For Address, key in the public IP address you want to use
   * For Virtual IP Password, set your own password
   * For VHID Group, select a vhid which you want to use
   * For Advertising Frequency, Base: 1 Skew: 0
   * For Description, enter the description for the CARP
   * Select save option, and CARP will be created

1. Create CARP for LAN interface
   * Repeat step 2
   * Change Interface to "LAN"
   * Change Address to a LAN IP you want to use

1. Set up HA synchronization on Firewall(1)
   * On Firewall(1), navigate to System >> High Availability >> Settings
   * Checked "Synchronize States" option
   * For Synchronize Interface, select "LAN" (not enough network interface, ideally to have a separate network for synchronization)
   * For Synchronize Peer IP, select the LAN IP of the other Firewall(2)
   * For Synchronize Config to IP, select the other Firewall(2) LAN IP also
   * For Remote System options, key in the username and password of the other Firewall(2) `root` account
   * Checked all the synchronize options boxes
   * Save settings and wait for the data to synchronize

1. Enable HA synchronization on Firewall(2)
   * On Firewall(2), navigate to System >> High Availability >> Settings
   * Checked Synchronize States option
   * For Synchronize Interface, select "LAN"
   * Save settings

1. Check the CARP status on both Firewall
   * Navigate to Firewall >> Virtual IPs >> Status
   * Firewall(1) will show "master" status
   * Firewall(2) will show "backup" status

1. CARP has been successfully set up
   * conduct testing on CARP by shutting down Master Firewall(1)
   * connection will be routed through Backup Firewall(2)

Done! HA has been successfully set up on Firewall.   

---
## Multi-Master Kubernetes Cluster
We will set up a multi-master K8s cluster with 3 master nodes and 5 worker nodes.
The setup is as such:   

|   Hostname   |   Node Type   |   IP Address   |   CPU Cores   |   RAM (GiB)   |   Hard Disk (GiB)   |
|---|---|---|---|---|---|
|   kube0   |   Master   |   10.0.1.100   |   2   |   4   |   64   |
|   kube1   |   Master   |   10.0.1.101   |   2   |   4   |   64   |
|   kube2   |   Master   |   10.0.1.102   |   2   |   4   |   64   |
|   kube3   |   Worker   |   10.0.1.103   |   2   |   4   |   128   |
|   kube4   |   Worker   |   10.0.1.104   |   2   |   4   |   128   |
|   kube5   |   Worker   |   10.0.1.105   |   2   |   4   |   128   |
|   kube6   |   Worker   |   10.0.1.106   |   2   |   4   |   128   |
|   kube7   |   Worker   |   10.0.1.107   |   2   |   4   |   128   |

|   Default Gateway   |   Name Server   |
|---|---|
|   10.0.1.5 (Firewall Virtual IP)   |   8.8.8.8   |


Install Kubernetes first (Refer to our [Github Page](https://github.com/CloudCommandos/missions/blob/master/orchestrate-containers/collab.md)).

You can follow the official guide to [form a multi-master k8s cluster](https://kubernetes.io/docs/setup/independent/high-availability/). Otherwise, to form a k8s cluster with stacked control plane nodes (etcd and master node components reside on the same nodes), create the following file `kubeadm-config.yaml`:
```yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "10.0.1.99:60443"  #<Virtual IP>:<any port not in use>
apiServer:
  certSANs:
   - "<cluster external IP>"
  extraArgs:
    advertise-address: "10.0.1.100"
networking:
  podSubnet: "10.244.0.0/16"  #Flannel subnet requirement
```
Then run the kubeadm init command as follows:
```bash
sudo kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs
```
Two join commands will be produced after the initialization. One for joining control planes (master nodes) into the cluster, and the other for joining worker nodes into the cluster.
Before joining in more nodes, the pod network of your choice should be implemented first.

Copy `/etc/kubernetes/admin.conf` out for permanent access for your user to kubectl:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

For our cluster we will go for Canal's pod network implementation:
```bash
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/canal/rbac.yaml
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/canal/canal.yaml
```

Now that we have formed our multi-master k8s cluster, we need to load balance API Server traffic among our master nodes. Recall the configuration in our `kubeadm-config.yaml`:
```yaml
controlPlaneEndpoint: "10.0.1.99:60443"
```
The `10.0.1.99` is a Virtual IP that should refer to all our master nodes collectively. To implement load balancing with this Virtual IP, we can use HAProxy and Heartbeat.

HAProxy is to implement port forwarding with endpoint health checks to load balance traffic among the master nodes. We will need more than one HAProxy running for high availability. For our setup we will be running HAProxy on each of our master nodes.
Install HAProxy
```bash
sudo apt-get install haproxy
```
Add the following to the end of `/etc/haproxy/haproxy.cfg`:
```yaml
frontend k8s-haproxy
    bind 10.0.1.99:60443
    mode tcp
    option tcplog
    default_backend k8s-haproxy

backend k8s-haproxy
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
        server kube0 10.0.1.100:6443 check fall 3 rise 2
        server kube1 10.0.1.101:6443 check fall 3 rise 2
        server kube2 10.0.1.102:6443 check fall 3 rise 2
```
Port `6443` is the default listening port for k8s API Server. You can change this in `/etc/kubernetes/manifests/kube-apiserver.yaml`. Just make sure that your HAProxy is configured to forward traffic to the correct destination port. Port `60443` is chosen as the binding port but you can choose any other ports that are not in use. Take note that our HAProxy resides on the master nodes, therefore `6443` is in use by default and therefore should not be used as the binding port.

Heartbeat is for the VMs that are running HAProxy to monitor each other's health such that at any point in time only one VM is bounded to the control plane endpoint Virtual IP `10.0.1.99`. This is to ensure as much as possible that the HAProxy logs are collected in order and are on the same VM.
Allow system services to bind non local IP on each HAProxy node:
```bash
nano /etc/sysctl.conf

net.ipv4.ip_nonlocal_bind=1
```
Then run
```bash
sysctl -p
```
Then start the HAProxy services
```bash
systemctl start haproxy
```

Install and enable Heartbeat on each HAProxy node
```
sudo apt-get -y install heartbeat && systemctl enable heartbeat
```
Now create a passkey for Heartbeat authentication
```
echo -n <any text> | md5sum

ThisIsTheGeneratedKey
```
Then create the file `/etc/ha.d/authkeys`:
```yaml
auth 1
1 md5 ThisIsTheGeneratedKey
```
Make sure that all the HAProxy nodes have a copy of this file.
This file also needs to be owned by root only
```bash
sudo chown root:root /etc/ha.d/authkeys
sudo chmod 600 /etc/ha.d/authkeys
```

Create the main configuration file `/etc/ha.d/ha.cf` for Heartbeat on all the HAProxy nodes. Change `ens18` to your network interface that the node can use to communicate with other HAProxy nodes.
```yaml
#       keepalive: how many seconds between heartbeats
#
keepalive 2
#
#       deadtime: seconds-to-declare-host-dead
#
deadtime 10
#
#       What UDP port to use for udp or ppp-udp communication?
#
udpport        694
mcast ens18 225.0.0.1 694 1 0
#       What interfaces to heartbeat over?
udp     ens18
#
#       Facility to use for syslog()/logger (alternative to log/debugfile)
#
logfacility     local0
#
#       Tell what machines are in the cluster
#       node    nodename ...    -- must match uname -n
node    kube0
node    kube1
node    kube2
```

Lastly create the `/etc/ha.d/haresources` file on all HAProxy nodes to specify the shared IP address and the preferred node to bind that IP.
```yaml
kube0 10.0.1.99
```
Restart all Heartbeat services
```bash
sudo systemctl restart heartbeat
```

---
## Set Up CephFS
CephFS is a convenient way to share files between servers and VMs. It is a mountable file system of Ceph. Containers can also use CephFS as mountable volume for data sharing or to make file changes dynamically from outside the container environment.

Proxmox VE v5.4 offers an intuitive and convenient way to set up CephFS. On the Proxmox web GUI, select one of the nodes then navigate the menu bar to 'CephFS' under 'Ceph'. Click on the 'Create CephFS' button to initialize CephFS with two auto-created pools. Then create one MDS (Metadata Server) for each node using the 'Create MDS' button. Assume the following setup for CephFS:

* CephFS
  |   Name   |   Data Pool   |   Metadata Pool   |
  | --- | --- | --- |
  |   cephfs   |   cephfs_data   |   cephfs_metadata   |

* MDS
  |   Name   |   Host   |   Address   |
  | --- | --- | --- |
  |   pve0   |   pve0   |   10.0.1.10:\<auto-assigned port>/******   |
  |   pve1   |   pve1   |   10.0.1.11:\<auto-assigned port>/******   |
  |   pve2   |   pve2   |   10.0.1.12:\<auto-assigned port>/******   |
  |   pve3   |   pve3   |   10.0.1.13:\<auto-assigned port>/******   |
  |   pve4   |   pve4   |   10.0.1.14:\<auto-assigned port>/******   |

### AutoFS to mount CephFS
We want to mount CephFS on our servers to conveniently distribute our project resources across VMs and Servers. If we mount CephFS through /etc/fstab for our servers, the result would be catastrophic since CephFS requires Ceph and that Ceph requires enough quorom to function properly. Since our servers are also the hosts of CephFS, mounting CephFS on bootup would fail since quorom will only be established after the bootup. Therefore we should use AutoFS to mount CephFS only on demand (which is only possible after the bootup).

#### On each of your servers:  
Use root account
```bash
su -
```
Extract the admin key of CephFS and save it into `/root/secrets/cephfs.key`
```bash
ceph auth get client.admin #copy the key from the output of this command
mkdir /root/secrets
nano /root/secrets/cephfs.key #paste the key inside this file
chmod -R 770 /root/secrets
```

Install AutoFS
```bash
sudo apt-get autofs
```

Add the following into `/etc/auto.master`
```yaml
/mnt    /etc/auto.cephfs  --timeout 60
```

Create the file `/etc/auto.cephfs` with the following contents:
```yaml
shares  -fstype=ceph,name=admin,secretfile=/root/secrets/cephfs.key,noatime             10.0.1.10:6789:/
```
The 'noatime' option tells CephFS not to save access time information of the mounted file system. This is to improve performance by preventing unnecessary write operations when files are read. Change the `10.0.1.10` ip address to the corresponding server's ip address accordingly.

Change directory to `/mnt/shares` to trigger the mount
```bash
cd /mnt/shares
```

For standardization, we also set up AutoFS to mount CephFS onto our VMs.


Useful Links:  
* [Configuring HA Kubernetes Cluster](https://medium.com/faun/configuring-ha-kubernetes-cluster-on-bare-metal-servers-with-kubeadm-1-2-1e79f0f7857b)

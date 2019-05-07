# Prototype Molly
We are to provision an operational Kubernetes (k8s) Cluster using 5 servers:
|   Server Name   |   CPU Cores   |   RAM (GiB)   |   Hard Disks   |
|---|---|---|---|
|   pve0   |   6   |   32   |   4 x 1.8TiB   |
|   pve1   |   6   |   32   |   4 x 1.8TiB   |
|   pve2   |   6   |   32   |   4 x 1.8TiB   |
|   pve3   |   6   |   32   |   3 x 1.8TiB   |
|   pve4   |   6   |   32   |   4 x 1.8TiB   |

## 1. Base Server Setup
Install Proxmox VE v5.4 on all 5 servers. You can use a flashed a USB thumb drive with the Proxmox image. We set up zfs RAID 1 using 2 hard disks on each server. For our servers, we have to change bootup mode and SATA config to legacy in order for zfs RAID to setup successfully. The Remaining 9 hard disks are used to set up Ceph. Ceph storage are to be used for the provisioning of Debian9.8 virtual machines. CephFS will also be set up to mount volume shares into k8s pods.

---
## 2. Internet Access Provisioning

---
## 3. Multi-Master Kubernetes Cluster
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
bcast  ens18
mcast ens18 225.0.0.1 694 1 0
ucast ens18 <current node IP>
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

Useful Links:  
* [Configuring HA Kubernetes Cluster](https://medium.com/faun/configuring-ha-kubernetes-cluster-on-bare-metal-servers-with-kubeadm-1-2-1e79f0f7857b)

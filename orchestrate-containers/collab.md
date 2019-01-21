# Container Orchestrator
Docker provides the platform to build, ship and run containers. However, what happens when you have multiple containers that you would like to run across multiple servers? Manually managing all the containers manually would be a tedious work. To resolve this issue, container orchestrator comes into play.

Container orchestrator automates the process of deployment, management, networking, scaling and availability of containers.

Some of the features that a container orchestrator have that Docker does not:

* Container scheduling
* Service discovery
* Redundancy and availability of containers
* Scaling up or down on the containers when required
* Monitoring the health status of the containers and host machines
* Migrate containers on a failed node to other available node
* Load Balancing

## Comparison on the Big Three Container Orchestrators
**Docker Swarm**
* Integrated with docker package version 1.12
* Easy to setup but limited customisation
* Suitable for small scale systems
* Flexible to change manager and worker nodes. Easy to promote or demote nodes between the roles
* Scaling has to be done with third-party application
* Yet to support external load balancing

**Kubernetes**
* Highly versatile and allow more customisation
* Web UI available
* Uses YAML or JSON for deployment
* Large open source community
* Require more details on setting up the cluster. Need to specific on the amount of nodes in the cluster and choose a master node among the cluster
* Auto-scaling
* Require less third-party software compared to Swarm and Mesos
* Flexible to deploy Kubernetes on any cloud or pre-existing data center

**Mesos**
* Good for large system and maximum redundancy. Able to run up to 50,000 nodes
* Used by companies like Airbnb and Twitter
* Require Marathon Framework for container orchestrator to work
* Uses Zookeeper to form High-Availability cluster
* Complexity and Flexibility

# Kubernetes
Kubernetes is the container orchestrator that automates the deployment, scaling and management of containers.

Some key features of Kubernetes are:

* Deploy multi-container application
* Provide networking, storage and service discovery
* Rolling on updates without any downtime

A standard Kubernetes setup consists of both master and worker nodes. A master node consists of 3 applications, API server, controller manager and scheduler. Kubernetes API uses command line `kubectl` to describe the desired state.

The worker node consists of 2 processes, kubelet and a proxy (kube-proxy). Kubelet is a "node agent" that starts and stops the pods and communicates with Docker Engine at a host level. Kubelet also communicates with API server on the master nodes and manages the containers, associated volumes and images. Kube-proxy reflects the kubernetes networking service on each node.

Common terms of Kubernetes:

Node: It a worker machine and it maybe either a physical machine or a virtual machine. A Master node controls each worker nodes. A node consists of at least a container runtime and Kubelet.

An example of container runtime is docker for pulling of image, unpacking and running the application. Kubelet is responsible for the communication between the master and worker nodes. It also manages the pods and containers running on the node.

Pod: A pod is a host application instances that can host a single or multiple containers and storage volumes. Pod has a unique IP address within its cluster and information on how to run container, container image version, specific ports to use etc. Data stored in the pod are not persistent and pod always runs on a node.

To list the pods, use `kubectl get pods`. `kubectl describe pods` will show the details on the pod's container such as container ID, IP address etc.

Although pods have IP addresses, but the IP addresses are not exposed outside of the cluster without a Service. Service is required to allow the application to receive traffic.

Here are some of the type of Services available:

1. ClusterIP - By default. This service is only reachable within the cluster.
1. NodePort - Expose the port of each selected node in the cluster via NAT. This makes the service accessible from the outside cluster using NodeIP:NodePort.
1. LoadBalancer - Create an external load balancer in the cloud and assigned it with a fixed external IP address to the service.


 To update container, `kubectl set image nameofdeployment container:version`

 To check the status of the rollout, `kubectl rollout status nameofdeployment`

 To undo a rollout, `kubectl rollout undo nameofdeployment`

 ## Installing Kubernetes
 Before installing Kubernetes, we must first install [Docker](https://docs.docker.com/install/linux/docker-ce/debian/#set-up-the-repository).

 To install Kubernetes enter the following commands:
```bash
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

Some of the minimum requirement to take note for installing Kubernetes:
1. 2 CPUs or more
1. 2GB or more of RAM
1. Ensure connectivity between the machine in the cluster
1. Swap must be disabled

To disable swap. Enter `swapoff -a` and this will disable swap but this is not persistent. Thus, to disable swap persistently, edit `/etc/fstab` and remove or comment out the line on swap.

Choose a pod network add-on before initialising the master node. The pod network is required for the pods to communicate with each other. A list of pod network is available [here](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

For this installation, Flannel will be used as the pod network. Thus, the command to initialise is
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=masteripaddresstocluster
```
The argument `--pod-network-cidr=10.244.0.0/16` is for Flannel setup.

`--apiserver-advertise-address=` is to be set with the master node IP adddress that has connection with the rest of the nodes in the cluster.

For Flannel, enter `sysctl net.bridge.bridge-nf-call-iptables=1` to pass bridged IPv4 traffic to  iptables chain.

After the initialisation is completed, remember to take down the command to add nodes to the master. The command should be display on the last line once the initialisation is completed. Example of the command with the token and cert:
```bash
kubeadm join 192.168.1.2:6443 --token 5dbj06.mbyb1suo8epe841a --discovery-token-ca-cert-hash sha256:0eb04182eea1061a0bd10a024b261e9eabd8e58e57bb99837938cd0c39138721
```
By default, the tokens expire after 24 hours. If the token expired, use the following command to create another token and print the join command:
```bash
kubeadm token create --print-join-command
```
To get the ca-cert, run the following command on the master node:
```bash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```   
For non-root user to run kubectl, enter the following commands:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
For root user, enter the following command:
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```
The above command can also resolve if error `The connection to the server localhost:8080 was refused - did you specify the right host or port?` is encountered.

Lastly, to finish up the configuration of the master node, we need to add the network driver. The driver should be selected during the `kubeadm init`. For this setup, Flannel is used and the command to add Flannel is:
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```

---
## Centralised Logging

To monitor individual nodes can be a hectic job. With a centralised logger, it easy for administrator to monitor all the nodes. One of the common logging stack in kubernetes is Elasticsearch, Fluentd and Kibana aka EFK.

EFK is comprised of:

* **Elasticsearch** is a distributed search and analytics engine, an object store where all logs are stored.
* **Fluentd** gathers logs from each nodes and forward it to Elasticsearch
* **Kibana** is a web UI to visualise the Elasticsearch data

Fluentd will be deployed on all the nodes in the cluster and aggregate the logs. Application in docker container writes stdout/stderr logs and stores in `/var/log/containers` directory.    
​Thereafter, Fluentd will forward the log to the node where Elaasticsearch is hosted. Kibana will access Elasticsearch data and provide a visualisation of the logs collected.

There are YAML files to configure the EFK stack in the Kubernetes GitHub [repository](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch).

You can either clone the whole repository down and run with `kubectl apply -f` or use `kubectl apply -f https://urloftheyaml` to run the YAML file for the EFK stack. In this case, the repository is clone to the master node.

Firstly, the first YAML to run is the `es-statfulset.yaml` file. Do take note that this script does not configure the logs to be stored in a persistent volume. Thus, the logs will be erased when the pods terminates.

To configure a **local** persistent volume, add the following script to the `es-statfulset.yaml` or a separate YAML file to create persistent volume. In this case, the script is added to the existing `es-statfulset.yaml` file. Add the script before the deployment of Elasticsearch.
Change the size of the storage to your own preference. As the `es-statfulset.yaml` application is created in the namespace `kube-system`, same namespace has to be used for the PersistentVolume and PersistentVolumeClaim. The hostPath define the directory on where the logs should be stored in the node.
```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  namespace: kube-system
  name: elasticsearch-volume
  labels:
    type: local
spec:
  storageClassName: elasticsearch
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data/elasticsearch"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: kube-system
  name: elasticsearch-pv-claim
  labels:
    app: elasticsearch-logging
spec:
  storageClassName: elasticsearch
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
```

Next, editing the following lines in the `es-statfulset.yaml`.
```yaml
volumes:
- name: elasticsearch-logging
  emptyDir: {}
```
and replace it the following:
```yaml
volumes:
- name: elasticsearch-logging
  persistentVolumeClaim:
    claimName: elasticsearch-pv-claim
```

Alternatively, if you are using **Ceph** for your storage, you can add the following lines. Change the IP address accordingly to your Ceph configurations.
```yaml
volumes:
- name: elasticsearch-logging
  cephfs:
    monitors:
      - 192.168.2.3:6789
      - 192.168.2.4:6789
      - 192.168.2.5:6789
    user: admin
    secretRef:
      name: cephfs-pass
    readOnly: false
    path: "/"
```

Once completed, run
```bash
kubectl apply -f es-statfulset.yaml
```
Run the following command to check the status of deployment of elasticsearch:
```bash
kubectl get pods -l k8s-app=elasticsearch-logging -n kube-system
```
If the deployment is successful, you should see the following if the replicas is set as 2:
```
NAME                      READY   STATUS    RESTARTS   AGE
elasticsearch-logging-0   1/1     Running   0          10s
elasticsearch-logging-1   1/1     Running   0          27s
```

Run the next YAML file to create a service for the elasticsearch-logging
```bash
kubectl apply -f es-service.yaml
```
Verify the deployment with the following:
```bash
kubectl get svc -l k8s-app=elasticsearch-logging -n kube-system
```

```
NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
elasticsearch-logging   ClusterIP   10.99.68.245   <none>        9200/TCP   44s
```

Next, let's deploy Fluentd as DaemonSet which will deploy it on all the nodes in the cluster. Note: The existing fluentd deployment YAML file from kubernetes GitHub repository seems to have some issue and the status of the deployment shows that there is a CrashLoopBackOff issue. More details on the issue is as shown below. Thus, another repository is been used to deploy fluentd.
```bash
kubectl logs fluentd-es-v2.2.1-8dr56 -n kube-system


2019-01-03 08:03:51 +0000 [error]: config error file="/etc/fluent/fluent.conf" error_class=Fluent::ConfigError error="Unknown filter plugin 'concat'. Run 'gem search -rd fluent-plugin' to find plugins"
```

Run the following command for fluentd configurations:
```bash
kubectl apply -f https://raw.githubusercontent.com/ecomm-integration-ballerina/efk-stack/master/fluentd-es-configmap.yaml
```

Side note: If you are using the kubernetes GitHub `fluentd-es-ds/yaml`, comment out the following lines to allow fluentd to be deployed on all nodes within the cluster:
```bash
nodeSelector:
  beta.kubernetes.io/fluentd-ds-ready: "true"
```

Run the following command:
```bash
kubectl apply -f https://raw.githubusercontent.com/ecomm-integration-ballerina/efk-stack/master/fluentd-es-ds.yaml
```

Verify the fluentd deployment with the following command:
```bash
kubectl get pods -n kube-system -l k8s-app=fluentd-es
```
If the deployment is successful, you should see the following for a cluster with 3 worker nodes:
```
NAME                      READY   STATUS    RESTARTS   AGE
fluentd-es-v2.2.0-8hmg4   1/1     Running   0          5s
fluentd-es-v2.2.0-qvsp5   1/1     Running   0          5s
fluentd-es-v2.2.0-vxchh   1/1     Running   0          5s
```

Lastly, lets deploy Kibana.
```bash
kubectl apply -f kibana-deployment.yaml
```

Verify the deployment with the following command:
```bash
kubectl get pods  -n kube-system -l k8s-app=kibana-logging
```
The output should be as of follows:
```
NAME                              READY   STATUS    RESTARTS   AGE
kibana-logging-764d446c7d-jp6gk   1/1     Running   0          2m29s
```

Now, run the `kibana-service.yaml` to deploy kibana service.
```bash
kubectl apply -f kibana-service.yaml
```
Verify the service deployment with the following command:
```bash
kubectl get svc  -n kube-system -l k8s-app=kibana-logging
```
The output of the above command should be:
```
NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kibana-logging   ClusterIP   10.109.171.94   <none>        5601/TCP   3s
```

Once the service is up, create an ingress yaml and name it `deployIngressKibana.yaml` and run the ingress yaml
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-kibana
  namespace: kube-system
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: node1
    http:
      paths:
      - path: /*
        backend:
          serviceName: kibana-logging
          servicePort: 5601
```

```bash
kubectl apply -f deployIngressKibana.yaml
```
Use `kubectl get ingress --all-namespaces` to check for the ingress deployed.


Now you can open up a browser and enter the your domain name specified in the `deployIngressKibana.yaml` host.

If you encounter a issue that shows `{"statusCode":404,"error":"Not Found","message":"Not Found"}` when you open up the site, comment off the following line in `kibana-deployment.yaml`.
```bash
- name: SERVER_BASEPATH
  value: /api/v1/namespaces/kube-system/services/kibana-logging/proxy
```

---
## Centralised Monitoring
Monitoring the kubernetes cluster is critical for the system administrator. Monitoring provides the platform to understand what is going on inside the system, the performance of the system, how many errors are there etc. One popular monitoring stack is the [Prometheus Stack](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus). Prometheus server will utilise service discovery in your cluster and connect to them and pull the metrics from your applications.


Here are some of the applications within the Prometheus monitoring stack:
1. Prometheus - Pull and storing of data in time-series format which are identified by metric name and key/value pairs
1. Grafana -  Provides web user interface for visualisation and monitoring dashboard for the data collected from Prometheus server
1. Alertmanager - Sends out notification via various channel such as email, to notify users of alerts.
1. Node-exporter - Collects system metrics such as cpu, memory, disk and expose them  where Prometheus server will pull the metrics

To start using Prometheus in your kubernetes cluster, `git clone https://github.com/coreos/prometheus-operator.git` the [Prometheus repository](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus) from Coreos GitHub to your master node.

change your directory to `prometheus-operator/contrib/kube-prometheus/`.

Enter the following to deploy Prometheus stack:
```bash
kubectl apply -f manifests/
```

Note: run the command twice if errors are shown during the deployment.

Run `kubectl get pods -n monitoring` to check the list of pods deployed for Prometheus stack. An example of `kubectl get pods -n monitoring` for a master and 3 worker nodes cluster shown below.
```
NAME                                   READY   STATUS    RESTARTS   AGE
alertmanager-main-0                    2/2     Running   0          23h
alertmanager-main-1                    2/2     Running   0          23h
alertmanager-main-2                    2/2     Running   0          23h
grafana-777cf74b98-tgt9s               1/1     Running   0          23h
kube-state-metrics-76b8dd6dd-44jn2     4/4     Running   0          23h
node-exporter-9dtvp                    2/2     Running   0          23h
node-exporter-cwdzp                    2/2     Running   0          23h
node-exporter-wzxmq                    2/2     Running   2          23h
node-exporter-xsp6t                    2/2     Running   0          23h
prometheus-adapter-66fc7797fd-pf92l    1/1     Running   0          23h
prometheus-k8s-0                       3/3     Running   1          23h
prometheus-k8s-1                       3/3     Running   1          23h
prometheus-operator-7df4c46d5b-gzqcs   1/1     Running   0          23h
```
Node-exporter is deployed as DaemonSet, which will be deployed in all nodes within the cluster.

Make sure all the pods in namespace `monitoring` are running. Next, Prometheus, grafana and alertmanager all have dashboard for web user interface. Do a quick port forwarding on your localhost to check if the applications are working with either your web browser or `curl` function.

To access Prometheus dashboard, enter the following:
```bash
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
```
To accesss Grafana dashboard, enter the following:
```bash
kubectl --namespace monitoring port-forward svc/grafana 3000
```
To accesss Alertmanager dashboard, enter the following:
```bash
kubectl --namespace monitoring port-forward svc/alertmanager-main 9093
```

Use `http://localhost:portnumber` to access the web interface or `curl http://localhost:portnumber` which will out put `<a href="/graph">Found</a>.` if deployment is successful. The port numbers for each applications are as followed:

| Application | Port Number |
| --- | --- |
| Prometheus | 9090 |
| Grafana | 3000 |
| Alertmanager | 9093 |

To deploy Prometheus stack with Ingress Controller, we need to create an ingress yaml. The `hostname/subdomainname` can be your hostname of your worker node in `/etc/hosts` or the sub-domain of your site.
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-pga
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: hostname/subdomainname
    http:
      paths:
      - path: /*
        backend:
          serviceName: prometheus-k8s
          servicePort: 9090
  - host: hostname/subdomainname
    http:
      paths:
      - path: /*
        backend:
          serviceName: grafana
          servicePort: 3000
  - host: hostname/subdomainname
    http:
      paths:
      - path: /*
        backend:
          serviceName: alertmanager-main
          servicePort: 9093
```

Now you can use the hostname/sub to access the web interface. You can now use Grafan to monitor the status of your kubernetes cluster!

If you would like to store the data into a persistent volume, head over [here](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/storage.md) to view the instructions.

---
# Kubernetes Workload - WordPress and MariaDB
## Set up Ceph Cluster
A persistent volume of type "local" is active on one worker node at a time, therefore data synchronization across pods on different nodes or during node migration is not possible without the use of 3rd party storage syncing solutions.
There are many available options but for this task we will use Ceph. You'll need at least 3 OSDs for Ceph to report a healthy status. Make sure that all your nodes can be accessed via ssh from the admin-node.  

On your Ceph Admin Node:  
Install Ceph-Deploy. Change 'jewel' to your Ceph release.
```bash
wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
echo deb https://download.ceph.com/debian-jewel/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
sudo apt update
sudo apt install ceph-deploy
```
Create Ceph-Cluster directory
```bash
mkdir my-ceph
cd my-ceph
```
Create the Ceph cluster
```bash
ceph-deploy new admin-node
```
Edit ceph.conf file and add in the ceph network
```bash
vim ceph.conf

public network = 10.142.10.0/24
```
Install Ceph packages on your Ceph cluster nodes
```bash
ceph-deploy install admin-node ceph-node-2 ceph-node-3
```
Deploy initial monitors and create Ceph keys
```bash
ceph-deploy mon create-initial
```
Copy configuration files and keys to the nodes
```bash
ceph-deploy admin admin-node ceph-node-2 ceph-node-3
```
Create OSDs. Make sure that the disks are at least 6GB.
```bash
ceph-deploy osd create admin-node:/dev/sdb
ceph-deploy osd create ceph-node-2:/dev/sdb
ceph-deploy osd create ceph-node-3:/dev/sdb
```
Create Meta-data server for CephFS
```bash
ceph-deploy mds create admin-node
```
Add Monitors
```bash
ceph-deploy mon add ceph-node-2
ceph-deploy mon add ceph-node-3
```
Check the Ceph Cluster
```bash
ceph quorum_status --format json-pretty
```

## Create CephFS
Create two storage pools
```bash
ceph osd pool create cephfs_data 128
ceph osd pool create cephfs_metadata 128
```
Create CephFS using the two pools
```bash
ceph fs new cephfs cephfs_metadata cephfs_data
```
Check that CephFS is up
```bash
ceph mds stat

#e10: 1/1/1 up {0=admin-node=up:active}
```

Obtain the ceph admin key from the admin-node. Copy only the key.
```bash
ceph auth get client.admin
```
When using root user to run kubectl commands
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```  
Store your CephFS password
```bash
kubectl create secret generic cephfs-pass --from-literal=key=YOUR_KEY
```  

## Deploy Ingress for port 80 and 443 traffic handling
Work in a new directory
```bash
mkdir ~/kubeproj
cd ~/kubeproj
```
Create Ingress deployment file `deployIngress.yml`
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: subdomain1.commandocloudlet.com
    http:
      paths:
      - path: /*
        backend:
          serviceName: wordpress
          servicePort: 80
```
We will create a Node Port listening on port 80/443 for our Ingress Controller. The default Node Port range is set to 30000-32767. Edit the `kube-apiserver.yaml` configuration file and overwrite the default range.
```bash
nano /etc/kubernetes/manifests/kube-apiserver.yaml

#add into spec -> containers -> command section
- --service-node-port-range=80-32767
```

Create Nginx Ingress Controller deployment file `deployIngressController.yml`. Nginx Ingress Controller routes traffic from port 80/443 to the defined service endpoints specified by Ingress.
```bash
curl -o deployIngressController.yml https://raw.githubusercontent.com/CloudCommandos/missions/master/orchestrate-containers/k8s%20files/deployIngressController.yml   
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      nodePort: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      nodePort: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
---
```


## Deploy WordPress and MariaDB in Separate Pods
Store your MariabDB and WordPress passwords
```bash
kubectl create secret generic mariadb-pass --from-literal=password=YOUR_PASSWORD
kubectl create secret generic wordpress-pass --from-literal=password=YOUR_PASSWORD
```  

Create MariaDB Deployment File `deployMariaDB.yml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mariadb
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mariadb
  clusterIP: None
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: wordpress-mariadb-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: wordpress
      tier: mariadb
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.244.0.0/16
    - podSelector:
        matchLabels:
          app: wordpress
          tier: frontend
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - ipBlock:
        cidr: 10.244.0.0/16
    ports:
    - protocol: TCP
      port: 3306
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: wordpress-mariadb
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
      tier: mariadb
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mariadb
    spec:
      containers:
      - image: mariadb:latest
        name: mariadb
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-pass
              key: password
        ports:
        - containerPort: 3306
          name: mariadb
        volumeMounts:
        - name: mariadb-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mariadb-persistent-storage
        cephfs:
          monitors:
            - 10.142.10.1:6789
            - 10.142.10.2:6789
            - 10.142.10.3:6789
          user: admin
          secretRef:
            name: cephfs-pass
          #secretFile: "/etc/ceph/user.secret"
          readOnly: false
          path: "/"
```  

Create WordPress Deployment File `deployWordPress.yml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
		resources:
          requests:
            cpu: "0.2"
          limits:
            cpu: "0.6"
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mariadb
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: wordpress-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        cephfs:
          monitors:
          - 10.142.10.1:6789
          - 10.142.10.2:6789
          - 10.142.10.3:6789
          user: admin
          secretRef:
            name: cephfs-pass
          readOnly: false
          path: "/"
```  

Deploy MariaDB and WordPress
```bash
cd ~/kubeproj
kubectl create -f deployMariaDB.yml
kubectl create -f deployWordPress.yml
```

You can now access your WordPress website via your sub-domain/public IP, e.g. `http://subdomain1.commandocloudlet.com`.

Also deploy a Horizontal Pod Autoscaler (HPA) for WordPress. HPA for a deployment will not work if there are no resource requests/limits set for the deployment.
The deployment file 'deployHPA.yml' is as such:
```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```
Deploy the HPA
```bash
kubectl apply -f deployHPA.yml
```

## Ansible scripts

Create an Ansible script to install docker and kubernetes packages on the nodes.

```bash
---
- hosts: all
  remote_user: root

  tasks:
    - name: install dependencies
      apt:
        name:
            - apt-transport-https
            - ca-certificates
            - curl
            - gnupg2
            - software-properties-common
            - python-pip
            - python-apt
            - openssl
        state: present
    - name: adding apt-key for docker
      apt_key:
        url: https://download.docker.com/linux/debian/gpg
        state: present

    - name: adding docker repo list
      apt_repository:
        repo: deb [arch=amd64] https://download.docker.com/linux/debian stretch stable
        state: present

    - name: install docker-ce and docker-compose
      apt:
        name:
            - docker-compose
            - docker-ce
        state: present

    - name: adding apt-key for kube
      apt_key:
        url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
        state: present

    - name: adding kube repo list
      apt_repository:
        repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
        state: present

    - name: install docker-ce and docker-compose
      apt:
        name:
            - kubeadm
            - kubectl
            - kubelet
        state: present

    - name: Commenting a line using the regualr expressions in Ansible.
      replace:
        path: /etc/fstab
        regexp: '(.*swap.*sw.*)'
        replace: '#\1'

    - name: Disable swap
      command: swapoff -a
      when: ansible_swaptotal_mb > 0
```

Create a script to set up the kubernetes cluster

```bash
---
- hosts: all
  remote_user: root

  vars_prompt:
    - name: "MasterIP"
      prompt: "Please enter the Master Node IP address that has connection with the rest of the nodes in the cluster (e.g 192.168.0.100)"
      default: "192.168.100.100"
      private: no

    - name: "SlaveName"
      prompt: "Please enter the actual hostname of the Slave Node (e.g node1)"
      default: "node1"
      private: no

    - name: "SlaveIP"
      prompt: "Please enter the Slave Node IP address that resides in the same network range as the Master Node (e.g 192.168.100.101)"
      default: "192.168.100.101"
      private: no

  tasks:
    - name: Check if etcd file exists
      stat: path=/etc/kubernetes/manifests/etcd.yaml
      register: etcd

    - name: Initialized the Master Node using canal as the pod network
      shell: "kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address={{MasterIP}}"
      when: etcd.stat.exists == False

    - name: Check if Kubernetes admin config exists
      stat: path=$HOME/.kube/config
      register: kubeconfig

    - name: Setup Master Node
      shell: |
        sysctl net.bridge.bridge-nf-call-iptables=1
        mkdir -p $HOME/.kube
        sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
      when: kubeconfig.stat.exists == False

    - name: Check if pod network has already been set up
      stat:
        path: /etc/cni/net.d/calico-kubeconfig
      register: podsetup

    - name: Setting up the pod network
      shell: |
        kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/canal/rbac.yaml
        kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/canal/canal.yaml
      when: podsetup.stat.exists == False

    - name: Checking cluster list for node
      shell: "kubectl get nodes"
      register: nodename

    - name: Status of Slave Nodes
      debug: msg="The specified node has already been added to the cluster or the hostname is already in used"
      when: nodename.stdout.find(SlaveName) != -1

    - name: Create new token for slave node to join cluster
      shell: "kubeadm token create --print-join-command"
      register: token
      when: nodename.stdout.find(SlaveName) == -1

    - name: Adding slave node into cluster
      shell: "ssh {{SlaveIP}} {{token.stdout}}"
      when: nodename.stdout.find(SlaveName) == -1

    - name: Printing of node joining msg
      debug: msg="Slave node with ip address {{SlaveIP}} has join the cluster using the command {{token.stdout}}"
      when: nodename.stdout.find(SlaveName) == -1
```
Run the scripts using the command `ansible-playbook 'scriptname'`. Do remember to input the target ip addresses into the script and also the ansible hosts file.

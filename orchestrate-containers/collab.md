# Container Orchestrator
Docker provides the platform to build, ship and run containers. However, what happen when you have multiple containers that you would like to run across multiple servers. Manually managing all the containers manually would be a tedious work. To resolve this issue, container orchestrator comes in play.

Container orchestrator automates the process of deployment, management, networking, scaling and availability of the containers.

Some of the features that a container orchestrator have that Docker doesn't.

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

The worker node consists of 2 processes, kubelet and a proxy (kube-proxy). Kubelet is a "node agent" that start and stop the pods and communicates with Docker Engine at a host level. Kubelet also communicate with API server on the master nodes and manages the containers, associated volumes and images. Kube-proxy reflects the kubernetes networking service on each node.

Common terms on Kubernetes:

Node: It a worker machine and it maybe either a physical machine or a virtual machine. A Master node controls each worker nodes. A node consists of at least a container runtime and Kubelet.

An example of container runtime is docker for pulling of image, unpacking and running the application. Kubelet is responsible for the communication between the master and worker nodes. It also manages the pods and containers running on the node.

Pod: A pod is a host application instances that can host a single or multiple containers and storage volumes. Pod has a unique IP address within its cluster and information on how to run container, container image version, specific ports to use etc. Data stored in the pod are not persistent and pod always runs on a node.

To list the pods, use `kubectl get pods`. `kubectl describe pods` will show the details on the pod's container such as container ID, IP address etc.

Although pods have IP addresses, but the IP addresses are not exposed outside of the cluster without a Service. Service is required to allow the application to receive traffic.

Here are some of the type of Service available:

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
```bash
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
```bash
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
```bash
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


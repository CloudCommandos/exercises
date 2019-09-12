I am not focusing on Ansible and Logging & Monitoring, so my observations are made on other services hosted in the non-prod VMs.

The major difference between our cluster and ESTL's non-prod cluster is that we are running most of our services in containers managed by Kubernetes, while ESTL uses Docker if the services are to run in containers.

This also contributed to our differences in running some of our services. For example, the non-prod cluster is running Certbot as a daemon to manage SSL certs. Our cluster deploys Cert-Manager inside Kubernetes to manage our SSL certs instead. Cert-Manager is a Kubernetes Operator so it can watch for Ingress objects to automate the registration/renewal of certs for individual web apps.
Certbot to manage certs. We are using cert-manager deployed in K8s.

logmon1 is running the Prometheus Stack in containers managed by Docker. Our setup is also using the same stack for monitoring but it is managed by Kubernetes. Since we are running Prometheus Operator, it is easier to setup monitoring for our K8s workloads. Young Ee will be able to share more on this.

core-svcs1 and core-svcs2 seems to be running the same services. Is the duplication meant for HA?

---
Below are some of the services running in the non-prod VMs of ESTL observed using ps command:
### Sampan
Sensu for monitoring.  
Uchiwa dashboard for Sensu.  
Cito Engine to receive alerts from Sensu.  
OpenVPN  
auditbeat  

### core-svcs1
OpenVPN  
dnsmasq  
auditbeat  
unbound  

### core-svcs2
OpenVPN  
dnsmasq  
auditbeat  
unbound  

### logmon1
Prometheus  
Grafana  
Alertmanager  
graylog  
mongodb  
fluentd  
elasticsearch  

### lb1
certbot  

### lb2
certbot-renew.sh  
nginx-periodic-reload.sh  

### common1
scripts running ssh ?  


## Questions
1. May we know what options are passed to the K8s master components for your production environment? E.g. your /etc/kubernetes/manifests/kube-apiserver.yaml etc if you used kubeadm to intialize your k8s cluster, or /etc/systemd/system/kube-apiserver.service if you went "The Hard Way".

2. If you went "The Hard Way", is there a script written for the initialization of your K8s cluster that we can look into?

3. How is K8s cluster upgrading handled?

4. Are customers allowed to access the K8s cluster? If so, what is the access topology like and how is it implemented? E.g. Management of shared services, private namespaces, K8s resources etc.

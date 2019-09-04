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



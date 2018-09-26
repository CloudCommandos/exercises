Build a Virtual Cluster
===
The bulk of today's IT workloads are run in virtual machines. These virtual machines are managed by a technology called a "hypervisor". The company that made VM technology mainstream is VMware and their hypervisor is ESXi. Opensource options are KVM, Xen, OpenVZ.

As a systems engineer, you will likely face the challenge of managing hundreds if not thousands of VMs. It is impossible to oversee so many machines without centralised control. Enterprise solutions such as VMware vSphere and Proxmox VE empowers the sysadmin to manage a large virtualised cluster from a single pane of glass.


The Task
---
1. Setup an account in one of the cloud service providers (eg. AWS, Azure, GCP)
1. Build a highly-available Proxmox VE (PVE) cluster of 3 nodes.
   1. If a node goes down, the downed workloads (ie. VMs) should migrate to a new node within reasonable time.
1. Using the config management tool Ansible, deploy the email server in the cluster. This time, install Postfix, Dovecot and Nginx in separate VMs. They should talk to each other over TCP/IP.
1. Document your steps in GitHub


Goals
---
The purpose of this exercise is to learn how to:
1. Setup a highly-available VM cluster
1. Deploy virtualised workloads
1. Use a config management tool to consistently deploy apps
1. Configure services to communicate over a TCP/IP network


Checkpoints
---
1. 5 Oct: Status update 1
1. 12 Oct: Status update 2
1. 19 Oct: End of exercise

May the odds be ever in your favour. Good luck!
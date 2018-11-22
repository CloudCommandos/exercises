Infra in Containers
===
The container is rapidly replacing the venerable VM as the virtualisation of choice. This exercise is for you to understand the differences between the two competing yet complementary technologies.


The Task
---
First, do some research to understand what containers are about.
1. Describe the differences between a VM and a container
1. Explain the pros and cons of using a container vs a VM
1. Talk about the different types of containers in the wild

You have gone through the rites of setting up the mail server on many levels - first in a single machine and then in the VM cluster. Let's do the same in containers to get a feel of the differences between managing VMs and containers.

Requirements
1. Compose your infra with Docker containers. Use `docker-compose` to organise the infra.
1. Each container should contain only 1 workload. Link them using Docker's network.
1. Make use of a config management tool (eg. Ansible, Terraform or both) to setup your infra.

Work together to deploy the infra in Google Cloud. The output of your work will be:
1. A working cluster of Docker containers that emulates the mailserver environment
1. Config management scripts to setup the cluster. I should be able to run these scripts myself to create a personal mail server.
1. A single document or a set of documents that elaborates your work
1. Post a writeup(s) on Medium.com which could be a subset of your documentation


Goals
---
The purpose of this exercise is to learn:
1. What containers are about
1. To setup a compute cluster based on Docker using config management tools


Checkpoints
---
1. 16 Nov: Status update 1
1. 23 Nov: Status update 2
1. 30 Nov: End of exercise


What's Next?
---
kubernetes is today the de facto container orchestrator. kubernetes is to containers what Proxmox VE is to VMs. To meaningfully manage hundreds and thousands of containers, you have to rely on an orchestrator to centrally manage your workloads.

Your next task will be to explore kubernetes. At the end of the next task, you'll move on to operate experimental workloads.

Bonne chance!

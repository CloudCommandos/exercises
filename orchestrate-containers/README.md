Orchestrate those Containers
===
Congrats! You have completed the containers exercise. Containers on their own are quite formidable but if you're tasked to manage hundreds of them by hand, you'll soon be admitted to a mental institution.

Enter the container orchestrator. kubernetes is the top dog when it comes to container management but it has many competitors that target the basic to advanced needs of the various IT teams.


The Task
---
First, research on the container orchestration tools out there.
1. Explain what a container orchestrator does that Docker Engine doesn't
1. Write a brief comparison of the various orchestrators in the wild
1. Explain how k8s controllers work under the hood
1. Describe how k8s launches a container and manages its lifecycle

You're probably bored of setting up the mail server so I'll spare you the pain :) This time, your job is to setup a highly-available Wordpress deployment and in addition, JIRA and Confluence. The latter 2 apps are what you'll use to raise change requests and build your internal team documentation. Learn to use them properly.

1. Setup k8s cluster using one of the public scripts
1. Incorporate centralised logging and monitoring
1. Install Wordpress and MariaDB (with HA) in separate pods. HA both of them.
1. Install Atlassian JIRA Core and learn how to use it for Change Requests. Buy the $10 license; I'll pay for it.
1. Install Atlassian Confluence and learn how to use it for team documentation. Buy the $10 license; I'll pay for it.
1. Firewall the ingress and egress of the pods
1. Test the HA and scaling features of k8s. Kill some pods and watch them ressurect.

Requirements
1. As far as possible, each container should contain only 1 running daemon. For eg., avoid running Wordpress and MariaDB in the same container.
1. Make use of a config management tool (eg. Ansible, Terraform or both) to setup your infra.

Work together to deploy the k8s cluster in Google Cloud. The output of your work will be:
1. A working k8s cluster
1. A single document or a set of documents that elaborates your work


Goals
---
The purpose of this exercise is to learn:
1. What container orchestration is about
1. To setup a compute cluster based on Docker and kubernetes using config management tools


Checkpoints
---
This is a particularly difficult mission to execute. Not only do you have to setup kubernetes and learn about it, you also have to bring online production-related services such as monitoring and logging.

1. 20 Dec: Status update 1
1. 31 Dec: Status update 2
1. 7 Jan: Status update 3
1. 14 Jan: Status update 4
1. 21 Jan: End of exercise


What's Next?
---
Jonathan is looking into bringing the heat strain project on board by end-Jan. Then you guys can pit your skills on a real production app!
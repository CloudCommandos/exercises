## Ansible

Comparison between ours and ESTL's ansible scripts shows a difference in terms of skill level based on the complexity of the scripts. 

The scripts we have at the moment is only to automate the creation of VM, install dependencies and also set up a kubernetes cluster. As compared to the scale of automation which ESTL's scripts are catered for, our scripts are very simple and offer less flexibility.

As our scripts are on a smaller scale, we can afford to consolidate all the required variables for the scripts into only one variable file. However for ESTL, the variables are split up into different categories, such as task variable, host variable & group variable.

Currently our automation scope is on a very small scale as compared to ESTL, which explains the difference in the complexity of our Ansible scripts. And also ESTL's scripts have been constantly tested and refined to achieve a professional standard so that it can be deployed in production environment.

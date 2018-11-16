## Containers instead of VM ##
With the many benefits of running Container to deploy applications as compared to VM, it proves to be a wise choice to swap the inner Vms with Containers. However, the choice to choose between Container and VM is based on what the user hopes to achieve.   
For this setup, Docker will be used to replace the inner VMs with Containers for the deployment of the applications and services.   
### Docker Installation ###
1. Setting up of repository.
  - Update the repository using `apt-get update`
  - Install the necessary packages:   
  ```
  apt-get install apt-transport-https ca-certificates curl gnupg2 software-properties-common
  ```
  - Adding Docker’s official GPG key:   
  ```
  curl -FsSL https://download.docker.com/linux/debian/gpg | apt-key add -
  ```
  - Set up the repository:   
  ```
  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cd) stable edge"
  ```
1. Install docker-ce & docker-compose package.
  - Run `apt-get update`
  - Run `apt-get install docker-ce docker-compose`
1. Verify docker-ce is installed correctly.
  - Run `docker run hello-world`
  - The command will download an image and runs it in a container. A message to show that the image is downloaded and run sucessfully will be printed.   

### Building Apache HTTP server container on Docker ###
1. Create a directory using `mkdir /docker`.
1. Create a Dockerfile inside the /docker directory; `touch /docker/Dockerfile`.
1. Create a sample webpage; `vim /docker/index.html`.
1. Edit Dockerfile to include;
```
FROM debian:stretch
MAINTAINER CloudCommando
RUN apt-get -y update
RUN apt-get -y install apache2
COPY index.html /var/www/html
CMD ["/usr/sbin/apache2ctl","-D","FOREGROUND"]
EXPOSE 80
```
1. Build an image using
```
docker build /docker/ -t apache2server:v1
```
1. Run the image with
```
docker run -d -p 8000:80 apache2server:v1
```
1. See the result by visiting http://HostIP:8000

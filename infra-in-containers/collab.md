## From VM to Container
### Differences between a VM and a Container
| VM | Container |
| ---- | ---- |
| Memory is allocated on setup | Memory is utilized on demand |
| Isolation is at hardware level (more secure) | Isolation is at virtual memory level (less secure) |
| Has individual OS libraries and kernel | Shares Host's OS libraries and kernel |
| Slower to run due to hardware emulation | Faster to run because they run directly on the host server |
| Does not have direct access to host's hardware | Has direct access to host's hardware |
| OS updates are done on each VM | OS updates are done on host |

### Pros and cons of using a Container vs a VM
Pros:  
1. Light weight (usually tens of MB)
1. Less resource intensive (no redundant OS services)
1. Faster startup
1. On-demand resource usage

Cons:  
1. Less secure due to isolation at a higher level
1. Only Containers of the same OS can be hosted on the same server (deployment constrain)
1. Isolation level constrained by networking requirements

### Different types of Containers in the wild
* LXC
* LXD
* Docker
* Solaris Zones
* RKT
* BSD Jails
* Windows Server Containers
* Hyper-V containers
* Kata


## Install Docker on Debian 9
Install the necessary packages for apt to be able to download packages over https.
```bash
sudo apt install apt-transport-https ca-certificates curl gnupg2 software-properties-common
```
Add GPG key for Docker repository
```bash
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```
Add Docker repository
```bash
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
```
Run update with the new repository
```bash
sudo apt update
```
Install Docker
```bash
sudo apt install docker-ce
```

## Install Docker Compose
Install Docker Compose with curl.
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
Give execute permission to the docker-compose binary.
```bash
sudo chmod +x /usr/local/bin/docker-compose
```
Create link file if running docker-compose gives the error
```bash
-su: /usr/bin/docker-compose: No such file or directory
```
```bash
ln -sf /usr/local/bin/docker-compose /usr/bin/docker-compose
```

## Docker User Guide
Here are some of the common docker commands:

To fetch a docker image from the Docker registry:

`docker pull imagethatyouwant`



To list the docker images in your system:

`docker images`

To run a docker image:

`docker run imagethatyouwanttorun`

Some of the common [options](https://docs.docker.com/engine/reference/commandline/run/) when running `docker run`:
* `-d` allow the container to run in detach mode. This option allow the container to run at the background and you can continue to use the terminal after the command.
* `-p port:port` exposed the port from the container to the external port. Example `-p 80:5000` Port 5000 used inside the container will be exposed to the external Port 80
* `-e` run the container with environment variable.

To list the current running container:

`docker ps`

add the option `-a` and this will show the running container and container that was ran. From the list, you should be able to see which container has exited or running.

To stop a running container:

`docker stop containerID`

The container ID is shown at the first column when running the command `docker ps` or `docker ps -a`.

To interact and run command in the container:

`docker run -it yourcontainer sh`

Once the container exited, try to always remove the container as it takes up disk space.

To remove container:

`docker rm containerID`

If you like to remove a list of exited containers without having to copy every single container ID, simply run this command:

`docker rm $(docker ps -a -q -f status=exited)`

Alternatively, you can run `docker container prune` to achieve the same result in removing all the exited containers.

To connect to a running container:

`docker exec -it containername/containerID /bin/bash`

With this command, it allows user to enter the bash shell in the container.

At times, environment variables are needed when running a container, and this environment variables can be stored in a file.

Create a file named `.env` and store the values of each environment variables that you want.

To run the container with environment variables:

`docker run --env-file=.env yourcontainer`

Most of information was referenced from [here](https://docker-curriculum.com).

### Building Customise image
Customisation can be done with the original image and with your own configuration. Configuration files from the local machine can be copy to the original image. This allow you to build a image with your own configuration.

To create a image, a `Dockerfile` is needed. Create a directory and store all the configurations along with the `Dockerfile` inside.

Store the configuration files in a sub directory to have a better organisation on the files.

let's build a image with [your own configuration with Nginx based image](https://www.nginx.com/blog/deploying-nginx-nginx-plus-docker/). An example of a `Dockerfile` is shown below.
```bash
FROM nginx
COPY content /usr/share/nginx/html
COPY conf /etc/nginx
```
* First line indicates the image that will be pull from either locally or docker hub
* The next line is to do a copy from the local folder `content` contents into the Nginx based image /usr/share/nignx/html. Note that the `content` and `conf` folder are both located within the same directory with Dockerfile

Next, run the following command to build your own image.

`docker build -t mynginx .`

The name of the image will be named as `mynginx` and the `.` is to indicate the current working directory with all the folders and Dockerfile.

The customised image should be built. Try to run this image and test if the container is working with your configuration.

### Push image to personal Docker Hub

The customised image can be push to a personal Docker Hub account. Thereafter, this will be available remotely for pulling. To do this, login into your docker hub account on your machine. The command to login is `docker login`. Enter your username and password when prompt.

You may also commit a container to an image which you ran previously. use `docker ps -a` to check the container ID of the container you want to commit. Next use the command `docker commit containerID imagename`. The new image will be created and can be verified by using `docker images`. You can also do `docker tag` to give a more detailed tag on the image.
Lastly, use `docker push imagename` and the image will be push to your personal Docker Hub.

# Spin off an Email Server with Containers
## Using Docker Hub Images and Docker Compose
A readily deployable email server suite is used.
The relevant docker images from public repository are:
1. tvial/docker-mailserver
1. hardware/rainloop
1. abiosoft/caddy

Docker-mailserver consists of Postfix, Dovecot, Spam Assassin etc. to form the email service, Rainloop serves as the web client for the email service, and Caddy is the web server.

Make a directory for your docker project
```bash
mkdir ~/dockerproj
```

Also make a directory for the email server suite under the docker project directory
```bash
mkdir ~/dockerproj/docker-mailserver
```

Pull the Container Images from Docker Hub into the email server suite directory
```bash
cd ~/dockerproj/docker-mailserver
docker pull tvial/docker-mailserver
docker pull hardware/rainloop
docker pull abiosoft/caddy
```

Also download tvial/docker-mailserver's configuration files
```bash
curl -o setup.sh https://raw.githubusercontent.com/tomav/docker-mailserver/master/setup.sh;
chmod a+x ./setup.sh;
curl -o docker-compose.yml https://raw.githubusercontent.com/tomav/docker-mailserver/master/docker-compose.yml.dist
curl -o .env https://raw.githubusercontent.com/tomav/docker-mailserver/master/.env.dist
```

Edit some of the environment parameters
```bash
vim ~/dockerproj/docker-mailserver/.env

  HOSTNAME=mail
  DOMAINNAME=commandocloudlet.com
  CONTAINER_NAME=mail
  SSL_TYPE=manual
```

Edit the email server suite's docker compose startup file
```bash
vim ~/dockerproj/docker-mailserver/docker-compose.yml

  version: '3'
  services:
    rainloop:
      image: hardware/rainloop
      links:
      - mail
      volumes:
      - ./data/rainloop:/rainloop/data
    mail:
      image: tvial/docker-mailserver:latest
      hostname: ${HOSTNAME}
      domainname: ${DOMAINNAME}
      container_name: ${CONTAINER_NAME}
      ports:
      - "25:25"
      - "143:143"
      - "465:465"
      - "587:587"
      - "993:993"
      - "4190:4190"
      volumes:
      - ./data/mail/data:/var/mail
      - ./data/mail/state:/var/mail-state
      - ./mail/config/:/tmp/docker-mailserver/
      - ./data/entry/ssl/mail.commandocloudlet.com:/tmp/ssl:ro
      environment:
      - DMS_DEBUG=${DMS_DEBUG}
      - ENABLE_CLAMAV=${ENABLE_CLAMAV}
      - ONE_DIR=${ONE_DIR}
      - ENABLE_POP3=${ENABLE_POP3}
      - ENABLE_FAIL2BAN=${ENABLE_FAIL2BAN}
      - ENABLE_MANAGESIEVE=${ENABLE_MANAGESIEVE}
      - OVERRIDE_HOSTNAME=${OVERRIDE_HOSTNAME}
      - POSTMASTER_ADDRESS=${POSTMASTER_ADDRESS}
      - POSTSCREEN_ACTION=${POSTSCREEN_ACTION}
      - REPORT_RECIPIENT=${REPORT_RECIPIENT}
      - REPORT_SENDER=${REPORT_SENDER}
      - REPORT_INTERVAL=${REPORT_INTERVAL}
      - SMTP_ONLY=${SMTP_ONLY}
      - SMTP_ONLY=${SMTP_ONLY}
      - SSL_TYPE=${SSL_TYPE}
      - SSL_CERT_PATH=/tmp/ssl/mail.commandocloudlet.com.crt
      - SSL_KEY_PATH=/tmp/ssl/mail.commandocloudlet.com.key
      - TLS_LEVEL=${TLS_LEVEL}
      - SPOOF_PROTECTION=${SPOOF_PROTECTION}
      - ENABLE_SRS=${ENABLE_SRS}
      - PERMIT_DOCKER=${PERMIT_DOCKER}
      - VIRUSMAILS_DELETE_DELAY=${VIRUSMAILS_DELETE_DELAY}
      - ENABLE_POSTFIX_VIRTUAL_TRANSPORT=${ENABLE_POSTFIX_VIRTUAL_TRANSPORT}
      - POSTFIX_DAGENT=${POSTFIX_DAGENT}
      - POSTFIX_MAILBOX_SIZE_LIMIT=${POSTFIX_MAILBOX_SIZE_LIMIT}
      - POSTFIX_MESSAGE_SIZE_LIMIT=${POSTFIX_MESSAGE_SIZE_LIMIT}
      - ENABLE_SPAMASSASSIN=${ENABLE_SPAMASSASSIN}
      - SA_TAG=${SA_TAG}
      - SA_TAG2=${SA_TAG2}
      - SA_KILL=${SA_KILL}
      - SA_SPAM_SUBJECT=${SA_SPAM_SUBJECT}
      - ENABLE_FETCHMAIL=${ENABLE_FETCHMAIL}
      - FETCHMAIL_POLL=${FETCHMAIL_POLL}
      - ENABLE_LDAP=${ENABLE_LDAP}
      - LDAP_START_TLS=${LDAP_START_TLS}
      - LDAP_SERVER_HOST=${LDAP_SERVER_HOST}
      - LDAP_SEARCH_BASE=${LDAP_SEARCH_BASE}
      - LDAP_BIND_DN=${LDAP_BIND_DN}
      - LDAP_BIND_PW=${LDAP_BIND_PW}
      - LDAP_QUERY_FILTER_USER=${LDAP_QUERY_FILTER_USER}
      - LDAP_QUERY_FILTER_GROUP=${LDAP_QUERY_FILTER_GROUP}
      - LDAP_QUERY_FILTER_ALIAS=${LDAP_QUERY_FILTER_ALIAS}
      - LDAP_QUERY_FILTER_DOMAIN=${LDAP_QUERY_FILTER_DOMAIN}
      - DOVECOT_TLS=${DOVECOT_TLS}
      - DOVECOT_USER_FILTER=${DOVECOT_USER_FILTER}
      - DOVECOT_PASS_FILTER=${DOVECOT_PASS_FILTER}
      - ENABLE_POSTGREY=${ENABLE_POSTGREY}
      - POSTGREY_DELAY=${POSTGREY_DELAY}
      - POSTGREY_MAX_AGE=${POSTGREY_MAX_AGE}
      - POSTGREY_AUTO_WHITELIST_CLIENTS=${POSTGREY_AUTO_WHITELIST_CLIENTS}
      - POSTGREY_TEXT=${POSTGREY_TEXT}
      - ENABLE_SASLAUTHD=${ENABLE_SASLAUTHD}
      - SASLAUTHD_MECHANISMS=${SASLAUTHD_MECHANISMS}
      - SASLAUTHD_MECH_OPTIONS=${SASLAUTHD_MECH_OPTIONS}
      - SASLAUTHD_LDAP_SERVER=${SASLAUTHD_LDAP_SERVER}
      - SASLAUTHD_LDAP_SSL=${SASLAUTHD_LDAP_SSL}
      - SASLAUTHD_LDAP_BIND_DN=${SASLAUTHD_LDAP_BIND_DN}
      - SASLAUTHD_LDAP_PASSWORD=${SASLAUTHD_LDAP_PASSWORD}
      - SASLAUTHD_LDAP_SEARCH_BASE=${SASLAUTHD_LDAP_SEARCH_BASE}
      - SASLAUTHD_LDAP_FILTER=${SASLAUTHD_LDAP_FILTER}
      - SASLAUTHD_LDAP_START_TLS=${SASLAUTHD_LDAP_START_TLS}
      - SASLAUTHD_LDAP_TLS_CHECK_PEER=${SASLAUTHD_LDAP_TLS_CHECK_PEER}
      - SASL_PASSWD=${SASL_PASSWD}
      - SRS_EXCLUDE_DOMAINS=${SRS_EXCLUDE_DOMAINS}
      - SRS_SECRET=${SRS_SECRET}
      - RELAY_HOST=${RELAY_HOST}
      - RELAY_PORT=${RELAY_PORT}
      - RELAY_USER=${RELAY_USER}
      - RELAY_PASSWORD=${RELAY_PASSWORD}
      cap_add:
      - NET_ADMIN
      - SYS_PTRACE
      restart: always

    entry:
      image: abiosoft/caddy:0.10.4
      restart: always
      privileged: true
      links:
        - rainloop
      ports:
        - "80:80"
        - "443:443"
      volumes:
        - ./entry/Caddyfile:/etc/Caddyfile
        - ./data/entry:/root/.caddy
```

Create Caddy's configuration Caddyfile
```bash
vim ~/dockerproj/docker-mailserver/entry/Caddyfile

  http://mail.commandocloudlet.com {
      proxy / rainloop:8888 {
          transparent
      }
  }
```

Create SSL Certificate
```bash
apt-get install openssl
mkdir -p ~/dockerproj/docker-mailserver/data/entry/ssl/mail.commandocloudlet.com
cd ~/dockerproj/docker-mailserver/data/entry/ssl/mail.commandocloudlet.com
openssl req -new -newkey rsa:4096 -x509 -sha256 -days 365 -nodes -out mail.commandocloudlet.com.crt -keyout mail.commandocloudlet.com.key
```

Start the Containers with Docker Compose
```bash
cd ~/dockerproj/docker-mailserver/
docker-compose up -d
```

If the containers are started successfully, you should see the following prompt
```bash
Starting mail ... done
Starting docker-mailserver_rainloop_1_d909ad2756f7 ... done
Starting docker-mailserver_entry_1_1a1132e80324    ... done
```

Check the postfix/dovecot container logs
```bash
docker logs -f mail
```

Create a user account for the Email Server
```bash
cd ~/dockerproj/docker-mailserver/
./setup.sh email add username@commandocloudlet.com password
```

## Configure Rainloop
Visit http://mail.commandocloudlet.com to access Rainloop email client.  
To configure Rainloop, visit http://mail.commandocloudlet.com/?admin.
The default root account is
```
Username: admin
Password: 12345
```
Change the password after login.

Add your domain using the admin control panel.
![Add domain settings into Rainloop](https://github.com/CloudCommandos/missions/blob/CC/infra-in-containers/screenshots/Rainloop_AddDomain.JPG)


## Setup Email Relay
Google Cloud Platform blocks outbound SMTP. Having an email relay server can help to overcome this limitaion.

## Current Status
Login into Rainloop web client is successful.  
Sending and receiving of emails are successful.

Useful Links:  
[How to Install and Use Docker on Debian 9](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-9)  
[Building a mailserver with modern webmail](https://www.davd.eu/byecloud-building-a-mailserver-with-modern-webmail/)


Let edit this sentence to try.

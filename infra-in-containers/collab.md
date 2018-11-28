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
1. Less secure due to isolation at a higher level (hybrid solution exists to overcome this, e.g. Kata)
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
Install Docker Compose
```bash
sudo apt-get install docker-compose
```

## Docker user guide
Here are some of the common docker commands:

To fetch a docker image from the Docker registry:
```bash
docker pull imagethatyouwant
```


To list the docker images in your system:
```bash
docker images
```

To run a docker image:
```bash
docker run imagethatyouwanttorun
```

To remove a docker image:   
```bash
docker image rm -f imagenameandtag
```

Some of the common [options](https://docs.docker.com/engine/reference/commandline/run/) when running `docker run`:
* `-d` allow the container to run in detach mode. This option allows the container to run in the background and you can continue to use the terminal after the command.
* `-p port:port` expose the port from the container to the external port. Example `-p 80:5000` Port 5000 used inside the container will be exposed to the external Port 80.
* `-e` run the container with environment variable.

To list the current running container:
```bash
docker ps
```

add the option `-a` and this will show the running containers and containers that was ran. From the list, you should be able to see which container has exited or is currently running.

To stop a running container:
```bash
docker stop containerID
```

The container ID is shown in the first column when running the command `docker ps` or `docker ps -a`.
To interact and run command in the container:
```bash
docker run -it yourcontainer sh
```

If your services are stateless, once their containers are exited, try to always remove the containers as they take up disk space.
To remove container:
```bash
docker rm containerID
```

If you would like to remove a list of exited containers without having to copy every single container ID, simply run this command:
```bash
docker rm $(docker ps -a -q -f status=exited)
```

Alternatively, you can run `docker container prune` to achieve the same result in removing all the exited containers.

This command allows the user to enter the bash shell of a running container.
```bash
docker exec -it containername/containerID bash
```

At times, environment variables are needed when running a container, and these environment variables can be stored in a file.

Create a file named `.env` and store the values of each environment variables that you want.

To run the container with environment variables:
```bash
docker run --env-file=.env yourcontainer
```

Most of the information are referenced from [here](https://docker-curriculum.com).

### Building custom image
Customisation can be done with the original image and with your own configuration. Configuration files from the local machine can be copied over to the original image. This allows you to build an image with your own configuration.

To create an image, a `Dockerfile` is required. Create a directory and store all the configurations along with the `Dockerfile` inside.

Store the configuration files in a sub directory to have a better organisation of the files.

let's build an image with [your own configuration with Nginx based image](https://www.nginx.com/blog/deploying-nginx-nginx-plus-docker/). An example of a `Dockerfile` is shown below.
```bash
FROM nginx
COPY content /usr/share/nginx/html
COPY conf /etc/nginx
```
* First line specifies the image that will be pulled either locally or from docker hub.
* The next line is to copy the contents from the local folder `content` into the Nginx-based image's /usr/share/nignx/html. Note that the `content` and `conf` folder are both located within the same directory as the Dockerfile.

Next, run the following command to build your own image.
```bash
docker build -t mynginx .
```
The name of the image will be named as `mynginx` and the `.` is to indicate the current working directory with all the folders and Dockerfile.  
The customised image should be built. Try to run this image and test if the container is working with your configuration.

### Push image to personal Docker Hub

The customised image can be pushed to a personal Docker Hub account. Thereafter, the image will be available remotely for pulling. To do this, login into your docker hub account on your machine. The command to login is `docker login`. Enter your username and password when prompted to do so.

You may also commit a container to an image that you had ran previously. use `docker ps -a` to check the container ID of the container that you want to commit. Next, use the command `docker commit containerID imagename`. The new image will be created and can be verified by using `docker images`. You can also use `docker tag` to give a more detailed tag on the image.
Lastly, use `docker push imagename` and the image will be pushed to your personal Docker Hub.

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
SSL_CERT_PATH=/tmp/ssl/ssl.crt
SSL_KEY_PATH=/tmp/ssl/ssl.key
```

Edit the email server suite's docker compose startup file
```bash
vim ~/dockerproj/docker-mailserver/docker-compose.yml

version: '2'
services:
  rainloop:
    container_name: rainloop
    image: hardware/rainloop
    links:
    - mail
    volumes:
    - rainloop_data:/rainloop/data
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
    - maildata:/var/mail
    - mailstate:/var/mail-state
    - ./config/:/tmp/docker-mailserver/
    - ./ssl:/tmp/ssl:ro
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
    - SSL_TYPE=${SSL_TYPE}
    - SSL_CERT_PATH=${SSL_CERT_PATH}
    - SSL_KEY_PATH=${SSL_KEY_PATH}
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
    container_name: entry
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
      - caddy_data:/root/.caddy
volumes:
  maildata:
    driver: local
  mailstate:
    driver: local
  rainloop_data:
    driver: local
  caddy_data:
    driver: local
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
mkdir -p ~/dockerproj/docker-mailserver/ssl
cd ~/dockerproj/docker-mailserver/ssl
openssl req -new -newkey rsa:4096 -x509 -sha256 -days 365 -nodes -out ssl.crt -keyout ssl.key
```

Create a user account for the Email Server
```bash
cd ~/dockerproj/docker-mailserver/
./setup.sh email add username@commandocloudlet.com password
```

Start the Containers with Docker Compose
```bash
cd ~/dockerproj/docker-mailserver/
docker-compose up -d
```

If the containers are started successfully, you should see the following prompt
```bash
Starting mail ... done
Starting rainloop ... done
Starting entry    ... done
```

Check the postfix/dovecot container logs
```bash
docker logs -f mail
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
Google Cloud Platform blocks outbound SMTP. Having an email relay server can help to overcome this limitation.

Create a free account on [sendgrid](https://sendgrid.com/). Follow the steps to create an api key and save the information.

Create the `./config/postfix-relaymap.cf` for relay host info
```bash
setup.sh relay add-domain <domainname> <relay-hostname> <port>
```

Create the `./config/postfix-sasl-password.cf` for relay authentication
```bash
setup.sh relay add-auth <domainname> <api-username> <api-password>
```

Add the relay hostname into the .env file
```bash
RELAY_HOST=smtp.sendgrip.net
```

Restart the containers for the new settings to take effect
```bash
docker-compose restart
```

Log on to [sendgrid](https://sendgrid.com/) and navigate to Sender Authentication tab under settings to authenticate your domain and also brand your links. Copy the information over to Godaddy (or your domain registrar of choice) and you should be able to send emails with your own domain branding.   

## Automate email server deployment with Ansible
Using Ansible, you can automate the deployment of the email server on a target server from a remote server.

On your remote server:  
Install Ansible
```bash
sudo apt-get install ansible
```

On your target server:  
Enable password login
```bash
nano /etc/ssh/sshd_config

PermitRootLogin yes
PasswordAuthentication yes
```

On your remote server:  
Generate ssh key pair and transfer the public key to the target server
```bash
ssh-keygen
ssh-copy-id <<target_server_ip>>
```

On your target server:  
Disable password authentication and enable public key authentication
```bash
nano /etc/ssh/sshd_config

PasswordAuthentication no
PubkeyAuthentication yes
```

On your remote server: 
Create an Ansible script file and fill in with the following contents
```bash
nano ~/deploy.yml

---
- hosts: all
  remote_user: root
  vars:
    base_path: /root/dockerproj/docker-mailserver
  vars_prompt:
    - name: "domain"
      prompt: "What is your domain name?"
      private: no
    - name: "emailurl"
      prompt: "What is your email client url?"
      private: no
    - name: "email_username"
      prompt: "We will be creating one email account. Please enter the login account id (e.g. user@example.com)"
      private: no
    - name: "email_password"
      prompt: "account password"
      private: yes
      confirm: yes
    - name: "relay_host"
      prompt: "What is your email relay host?"
      private: no
    - name: "relay_port"
      prompt: "What is your email relay port?"
      private: no
    - name: "relay_username"
      prompt: "What is your email relay username?"
      private: no
    - name: "relay_key"
      prompt: "What is your email relay password?"
      private: no

  tasks:
    - name: Ensure {{base_path}} exists
      file: path={{base_path}} state=directory
    - name: "download docker-compose.yml"
      get_url:
        url: https://raw.githubusercontent.com/CloudCommandos/missions/CC/infra-in-containers/resources/docker-compose.yml
        dest: "{{base_path}}/docker-compose.yml"
        mode: 0777
        force: yes
    - name: "download .env"
      get_url:
        url: https://raw.githubusercontent.com/CloudCommandos/missions/CC/infra-in-containers/resources/env.txt
        dest: "{{base_path}}/.env"
        mode: 0777
        force: yes
    - name: "download setup.sh"
      get_url:
        url: https://raw.githubusercontent.com/CloudCommandos/missions/CC/infra-in-containers/resources/setup.sh
        dest: "{{base_path}}/setup.sh"
        mode: 0777
        force: yes

    - name: "edit domain name in .env"
      lineinfile:
        dest: "{{base_path}}/.env"
        regexp: '^DOMAINNAME='
        line: 'DOMAINNAME={{domain}}'
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
    - name: install pexpect
      pip:
        name: pexpect
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
    - name: create directory
      file:
        path: "{{base_path}}/ssl"
        state: directory
    - name: generate a self signed OpenSSL certificate
      expect:
        command: openssl req -new -newkey rsa:4096 -x509 -sha256 -days 365 -nodes -out {{base_path}}/ssl/ssl.crt -keyout {{base_path}}/ssl/ssl.key
        responses:
             "Country Name" : ""
             "State or Province Name" : ""
             "Locality Name" : ""
             "Organization Name" : ""
             "Organizational Unit Name" : ""
             "Common Name" : ""
             "Email Address" : ""
    - name: Ensure {{base_path}}/entry exists
      file: path={{base_path}}/entry state=directory
    - name: "create Caddy file"
      copy: 
        content: "{{emailurl}} { {{'\n'}}proxy / rainloop:8888 { {{'\n'}}transparent {{'\n'}} } {{'\n'}} }"
        dest: "{{base_path}}/entry/Caddyfile"
    - name: "setup first email account"
      shell: "./setup.sh email add {{email_username}} {{email_password}}"
      args:
        chdir: "{{base_path}}/"
      ignore_errors: yes
    - name: "setup email relay domain"
      shell: "./setup.sh relay add-domain {{domain}} {{relay_host}} {{relay_port}}"
      args:
        chdir: "{{base_path}}/"
      ignore_errors: yes
    - name: "setup email relay auth"
      shell: "./setup.sh relay add-auth {{domain}} {{relay_username}} {{relay_key}}"
      args:
        chdir: "{{base_path}}/"
      ignore_errors: yes

    - name: "start docker containers"
      docker_service:
        project_src: "{{base_path}}"
```

Run the script
```bash
ansible-playbook -i <<target_server_ip>>, ~/deploy.yml
```

You should see the following prompts
```bash
PLAY [all] *********************************************************************

TASK [setup] *******************************************************************
ok: [target_server_ip]

TASK [Ensure /root/dockerproj/docker-mailserver exists] ************************
changed: [target_server_ip]

TASK [download docker-compose.yml] *********************************************
changed: [target_server_ip]

TASK [download .env] ***********************************************************
changed: [target_server_ip]

TASK [download setup.sh] *******************************************************
changed: [target_server_ip]

TASK [edit domain name in .env] ************************************************
ok: [target_server_ip]

TASK [install dependencies] ****************************************************
changed: [target_server_ip]

TASK [install pexpect] *********************************************************
changed: [target_server_ip]

TASK [adding apt-key for docker] ***********************************************
changed: [target_server_ip]

TASK [adding docker repo list] *************************************************
changed: [target_server_ip]

TASK [install docker-ce and docker-compose] ************************************
changed: [target_server_ip]

TASK [create directory] ********************************************************
changed: [target_server_ip]

TASK [generate a self signed OpenSSL certificate] ******************************
changed: [target_server_ip]

TASK [Ensure /root/dockerproj/docker-mailserver/entry exists] ******************
changed: [target_server_ip]

TASK [create Caddy file] *******************************************************
changed: [target_server_ip]

TASK [setup first email account] ***********************************************
changed: [target_server_ip]

TASK [setup email relay domain] ************************************************
changed: [target_server_ip]

TASK [setup email relay auth] **************************************************
changed: [target_server_ip]

TASK [start docker containers] *************************************************
changed: [target_server_ip]

PLAY RECAP *********************************************************************
target_server_ip              : ok=19   changed=17   unreachable=0    failed=0
```


Useful Links:  
[How to Install and Use Docker on Debian 9](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-9)  
[Building a mailserver with modern webmail](https://www.davd.eu/byecloud-building-a-mailserver-with-modern-webmail/)

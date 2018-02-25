# Moving Jenkins infrastructure into Docker
## Purpose
This article details how to convert a standard Jenkins infrastructure into a Docker environment with the following features:
*	Jenkins runtime and configuration in separate containers for easy upgrade/rollback
*	Existing Jenkins config imported into Docker environment
*	Multiple Jenkins Docker environments (e.g. production and test) running simultaneously

The main benefit of using Docker is for its lightweight footprint, better performance and portability.
## Infrastructure
The following components are used:
*	Vagrant
*	Docker, Docker Compose
*	Jenkins
## Image
Install Vagrant and Virtual Box (or VMWare) on Windows.

Then run the following in a Windows Command Prompt:

```
cd \
mkdir VagrantImage
cd VagrantImage
vagrant init centos/7
```
A Centos 7 image is used in this instance.

Enable the following in the *Vagrantfile*:
```
config.vm.network "forwarded_port", guest: 8080, host: 8080
config.vm.network "public_network"
```

Start up the image:
```
vagrant up
```
## Install Docker
Access the shell:
```
vagrant ssh
```
I prefer to install known working infrastructures instead of using the latest software.

Install Docker:
```
sudo yum install docker-1.12.6-68.gitec8512b.el7.centos
```
Run the following to manage Docker as a non-root user:
```
sudo groupadd docker
sudo usermod -aG docker vagrant
```
For the changes to apply, exit the shell and restart the image:
```
vagrant halt
vagrant up
```
Access the shell again and start the Docker service:
```
vagrant ssh
sudo service docker start
```
Show the Docker environment:
```
docker info
```
## Container for Jenkins config
Create a *Dockerfile* in a *jenkins-data* directory for the data volume.
```
# Use the base Debian image because it matches the same base image the Cloudbees Jenkins image uses.
# Because we’ll be sharing file systems and UID’s across containers we need to match the operating systems.
FROM debian:jessie

MAINTAINER Nigel Robbins

# Create the Jenkins user in this container
RUN useradd -d "/var/jenkins_home" -u 1000 -m -s /bin/bash jenkins

# Recreate the Jenkins log directory
RUN mkdir -p /var/log/jenkins
RUN chown -R jenkins:jenkins /var/log/jenkins

# Add the Docker volume
VOLUME ["/var/log/jenkins","/var/jenkins_home"]

# Set the user of this container to “jenkins” for consistency
USER jenkins

CMD ["echo", "Data container for Jenkins"]
```
Build the image for the Jenkins data volume:
```
docker build -t jenkinsdata jenkins-data/.
```
Start the Jenkins data volume container:
```
docker run --name=jenkins-data jenkinsdata
```
## Container for Jenkins runtime
Create a *Dockerfile* in a *jenkins-master* directory for the Jenkins runtime.
```
# This is the base image we use to create our image from
FROM jenkins:1.651.3

MAINTAINER Nigel Robbins

# Create required directories and give access to jenkins user
USER root
RUN mkdir /var/log/jenkins
RUN mkdir /var/cache/jenkins
RUN chown -R jenkins:jenkins /var/log/jenkins
RUN chown -R jenkins:jenkins /var/cache/jenkins
USER jenkins

# Set the list of plugins to download/update in plugins.txt like this:
# pluginID:version
# periodicbackup:1.3
# ...
# NOTE : Only set pluginID to download latest version of the plugin
COPY plugins.txt /usr/share/jenkins/plugins.txt

# Set the default port but allow to be overriden via a build parameter
ARG HTTP_PORT=8080
# Environment variables
ENV HTTP_PORT=${HTTP_PORT}
ENV JENKINS_OPTS="--httpPort=$HTTP_PORT --handlerCountStartup=100 --handlerCountMax=300 --logfile=/var/log/jenkins/jenkins.log --webroot=/var/cache/jenkins/war"
```
Note that plugins to be installed can be detailed in the *jenkins-master/plugins.txt* file.

Build the image for the Jenkins runtime:
```
docker build -t jenkinsmaster jenkins-master/.
```
Start the Jenkins runtime container referencing the Jenkins data volume container:
```
docker run -p 8080:8080 -p 50000:50000 --name=jenkins-master --volumes-from=jenkins-data -d jenkinsmaster
```
Run the following:
```
docker ps -a
```
This will show the running *jenkins-master* container and the stopped *jenkins-data* container which is expected for volume containers.

The following command will display the newly created images:
```
docker images
```
Start using Jenkins in a browser:

![image](https://user-images.githubusercontent.com/18073204/36062206-23009572-0e5f-11e8-94e0-16c0d89ce32a.png)

Go to **Manage Jenkins** and then **Manage Plugins** where you will see the plugins installed as detailed in *plugins.txt*.

## Persistent data
In the Jenkins UI, create a job and build it.

Stop/remove the *jenkins-master* container, then start it up again.
```
docker stop jenkins-master
docker rm jenkins-master
docker run -p 8080:8080 -p 50000:50000 --name=jenkins-master --volumes-from=jenkins-data -d jenkinsmaster
```
In the Jenkins UI, you will see the job previously created along with its **Build History**.
## Easy upgrade and rollback
To upgrade Jenkins, simply change the Jenkins version in the *Dockerfile* and/or the plugins in the *plugin.txt* and rebuild the image.

Then, stop/remove the Jenkins container and start it up again.

To roll back, stop/remove the container, rebuild the old image and start up the container.

Alternatively, have a different *jenkins-master* directory/image for a different version (e.g. *Jenkins-master-2.1.50*).
## Importing existing Jenkins config
Unless you are starting from scratch, it’s likely you’ll already have Jenkins configuration in an existing environment.

It’s normal for the Jenkins config to be backed up on a regular basis (e.g. via the *periodic backup* plugin).

The Cloudbees image that is referenced in the *jenkins-master/Dockerfile* uses a *jenkins.sh* script to configure and start up Jenkins. The *jenkins.sh* script will need to be modified to import your existing Jenkins configuration.

Copy the script from the container:
```
docker cp jenkins-master:/usr/local/bin/jenkins.sh jenkins-master/jenkins.sh
```
Add the following lines to the script before the *exec java* call:
```
cp /usr/share/jenkins/ref/jenkins_config_backup.tar.gz /var/jenkins_home
cd /var/jenkins_home
gunzip jenkins_config_backup.tar.gz
tar -xvf jenkins_config_backup.tar
```
Change the *jenkins-master/Dockerfile* and add the following before the *ENV* statements:
```
COPY jenkins.sh /usr/local/bin/jenkins.sh
COPY jenkins_config_backup.tar.gz /usr/share/jenkins/ref
```
Copy the existing Jenkins config to be imported:
```
cp jenkins_config_backup.tar.gz jenkins-master/jenkins_config_backup.tar.gz
```
Stop/remove the *jenkins-master* container, rebuild the image, and start up the container.
```
docker stop jenkins-master
docker rm jenkins-master
docker build -t jenkinsmaster jenkins-master/.
docker run -p 8080:8080 -p 50000:50000 --name=jenkins-master --volumes-from=jenkins-data -d jenkinsmaster
```
Your existing Jenkins config can now be seen in the Jenkins UI.

If required, the **Build History** of the various jobs can also be imported.
##	Multiple Jenkins Docker environments
It is good practice to make changes (e.g. modify/create jobs, upgrade Jenkins master version, add plugins, etc.) in a test environment before rolling into production.

A test environment using a different port can be set up by creating a test master image based on the production:
```
cp -r jenkins-master jenkins-master-test
```
The *Dockerfile* is set up so that the port can be changed when building the image. For example:
```
docker build --build-arg 'HTTP_PORT=9090' -t jenkinsmastertest jenkins-master-test/.
```
The same *jenkins-data* image can be in the test environment.

Start the container giving it a different name:
```
docker run --name=jenkins-data-test jenkinsdata
```
Start the Jenkins test runtime container referencing the Jenkins test data volume:
```
docker run -p 9090:9090 --name=jenkins-master-test --volumes-from=jenkins-data-test -d jenkinsmastertest
```
Note that the port will need to be enabled in the *Vagrantfile* and the image restarted:
```
config.vm.network "forwarded_port", guest: 9090, host: 9090
```
## Useful docker commands
Tail the Jenkins log in the container:
```
docker exec jenkins-master tail -f /var/log/jenkins/jenkins.log
```
List/display files in a container:
```
docker exec jenkins-master ls /usr/local/bin/jenkins.sh
docker exec jenkins-master ls /var/jenkins_home/plugins
docker exec jenkins-master ls /var/jenkins_home/jobs
docker exec jenkins-master cat /var/jenkins_home/config.xml
```
Remove an image:
```
docker rmi jenkinsmaster
```
Get help on Docker commands:
```
docker --help
```
## Docker Compose
Docker Compose is a tool for defining and running multi-container Docker applications.

It eases the management of images and containers.

Install Docker Compose:
```
sudo curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
Create a *docker-compose.yml* file in the root directory:
```
jenkinsdata:
  build: jenkins-data
jenkinsmaster:
  build: jenkins-master
  volumes_from:
    - jenkinsdata
  ports:
    - "8080:8080"
    - "50000:50000"
```
Build the images:
```
docker-compose build
```
Start the containers:
```
docker-compose up -d
```
Show the running containers:
```
docker-compose ps
```
Note that by default, Docker Compose names the images with a prefix based on the parent directory (*vagrant* in this case).

![image](https://user-images.githubusercontent.com/18073204/36062196-efd6a75e-0e5e-11e8-9e42-0f16f2f12185.png)

Stop the containers:
```
docker-compose stop
```
Remove the Jenkins master container:
```
docker-compose rm jenkinsmaster
```
The command below will remove all containers (including the data container so be aware that any changes will be lost):
```
docker-compose rm
```
## Useful tips
To get a list of installed plugins in a Jenkins master, go to the Script Console and run the following:
```
Jenkins.instance.pluginManager.plugins.each{
  plugin ->
    println ("${plugin.getShortName()}:${plugin.getVersion()}")
}
```
A list of the available Jenkins base images can be found at [https://hub.docker.com/]. Search for Jenkins and go to the **Tags** folder and click **View Unscanned Tags**.

Configure Docker to start on boot:
```
sudo systemctl enable docker
```
List available docker versions:
```
sudo yum list docker.x86_64 --showduplicates | sort –r
```
Check installed docker versions:
```
docker --version
docker-compose --version
rpm -qa | grep -i docker
```
Uninstall docker and docker compose:
```
sudo yum remove docker-1.12.6-68.gitec8512b.el7.centos.x86_64
sudo yum remove docker-common-1.12.6-68.gitec8512b.el7.centos.x86_64
sudo rm -f /usr/local/bin/docker-compose
```
## Enhancements
It is recommended to include a proxy container (e.g. Nginx) to enforce things like redirects to HTTPS and to mask Jenkins listening on port 8080 with a web server that listens on port 80.

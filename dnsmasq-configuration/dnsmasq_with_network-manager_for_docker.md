# dnsmasq (Network Manager) for Docker containers 
## Background
In trying to get this dnsmasq to work with docker I tried several methods.  In the **dnsmasq_wrapper_alternative** folder is one is based on the one found here: [Using Dnsmasq with Ubuntu for VM web application testing](https://gist.github.com/magnetikonline/6236150).  Currently this works as long as you have an active network connection that is managed by Network Manager. A fix for those times that you do not have a network connection is on the todo list.

## Setup
Host computer is running Ubuntu 14.04.

## Using a configuration file
Using information from a comments that  [Bozzie4](https://gist.github.com/Bozzie4)  and [harish2704](https://gist.github.com/harish2704) made to [magnetikonline](https://gist.github.com/magnetikonline) gist. I removed all the options that I could and put them in configuration file to allow me to override the hardcoded Network Manager dnsmasq configuration.

## The Issue
I have not found a way to override the hardcoded **--listen-address=127.0.1.1** that Network Manager sends to dnsmasq. This means that all other IP addresses that you want dnsmasq to listen on must be specifically given in the configuration. 

Using the alternative wrapper method it is possible to use a configuration without any **--listen-address=** line and have dnsmasq listen on all IP adresses. 

## Almost default docker0 interface IP address
I have included in the configuration file the line:
`listen-address=172.17.42.1` If this address is not in use on your computer Docker will use it with the psuedo network adapter **docker0**. 
This will allow your containers to use this ip addres as a dns server. 

## DNS for .docker
The **dockerhostdns.tmpl** in the **templates** folder will configure the almost default ip address of 172.17.42.1 as the destination for all **.docker** hostnames 

## DNS for containers
The **dnsmasq.tmpl** in the **templates** folder will create a dns host file for dnsmasq.

## docker-compose.yml
### Volumes to mount

- `"/var/run/docker.sock:/tmp/docker.sock"`
- `"~/units/dockergen/:/etc/docker-gen/templates"` **- docker-gen templates**
- `"./docker-gen.conf:/etc/docker-gen/conf.d/docker-gen.conf"` **- docker-gen configuration file**
- `"/var/tmp/dockerhosts:/var/tmp/dockerhosts"` **- where the dnsmasq hostname files will be created**

### Command to run
We can override the run command for the docker-gen container by adding this line to our docker-compose.yml. Her we are telling the container to use our configuration file that we mounted above.
 ```command: -config /etc/docker-gen/conf.d/docker-gen.conf```

### More Info
See the [docker-compose-localdevmeos](https://github.com/meosch/docker-compose-localdevmeos) project for more info. Specifically the **dockergen:** container **volumes:**, **command:** sections of the **docker-compose_all_containers.yml** file.
Also see the **docker-gen.conf** file from the same project for an example of a docker-gen configuration file.

## Docker Configuration
To enable the containers to use the host computers DNS provided by dnsmasq it is adviseable to set the IP address for the pesudo network adapter **docker0** to **172.17.42.1** and also set the DNS server to use for lookups to the same. Details follow in the installation section.

**Note:** For Ubuntu 15.04 the place to configuration for Docker has changed. See [Setting Docker’s DOCKER_OPTS on Ubuntu 15.04](http://blog.benhall.me.uk/2015/07/setting-dockers-docker_opts-on-ubuntu-15-04/)

## Installation
### dnsmasq
* Copy the **dnsmasq.conf** in this folder to **/etc/NetworkManager/dnsmasq.d/dnsmasq.conf** (from this folder run) 

```cp dnsmasq.conf /etc/NetworkManager/dnsmasq.d/dnsmasq.conf```

* Create links to the dnsmasq host files that docker-gen will create with:

```sudo ln -s -T /var/tmp/dockerhosts/dockerhosts /etc/NetworkManager/dnsmasq.d/dockerhosts
sudo ln -s -T /var/tmp/dockerhosts/docker /etc/NetworkManager/dnsmasq.d/docker ```

* Create the directory and empty files for dnsmasq to use for configuration when docker-gen has not yet mount. Otherwise dnsmasq will complain that the configuration files do not exist, not start and break your networking.
```mkdir /var/tmp/dockerhosts
touch /var/tmp/dockerhosts/docker
touch /var/tmp/dockerhosts/dockerhosts```

* Restart Network Manager with 
```sudo restart network-manager```

### docker
* Copy **docker** to **/etc/default/docker** (if you don't have this file) or add these two lines to your current **/etc/default/docker** file:


<pre>
    <strong># Always use the same ip address and also use it for dns
    DOCKER_OPTS="--bip=172.17.42.1/24 --dns=172.17.42.1"</strong></pre>


* Restart Docker with
`sudo restart docker`

## Accessing new containers
As you bring up new containers or restart them you will need to get dnsmasq to reload the docker container DNS files that were created by docker-gen. You can do this by restarting Network Manager with
`sudo restart network-manager`

## Docker Container access from other computers
If you need to access to your docker containers from other computers you will need use the dnsmasq wrapper method. See **dnsmasq_wrapper_alternative.md**  in the **dnsmasq_wrapper_alternative** folder.

## Todo
* Currently if you do not have a network connection via Network Manager  that is active dnsmasq does not work or return an answer. Find a way to setup a dummy network adapter that is managed by Network Manger to use in this cases.

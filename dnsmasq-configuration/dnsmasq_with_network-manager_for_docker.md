# dnsmasq (Network Manager) for Docker containers 
## Background
In trying to get this dnsmasq to work with docker I tried several methods.  In the **dnsmasq_wrapper_alternative** folder is one is based on the one found here: [Using Dnsmasq with Ubuntu for VM web application testing](https://gist.github.com/magnetikonline/6236150).  Currently this works as long as you have an active network connection that is managed by Network Manager. A fix for those times that you do not have a network connection is on the todo list.

## Setup
Host computer is running Ubuntu 14.04.3 LTS , Trusty.

## Docker IP Address Configuration
#### The almost default docker0 interface IP address . . .
If the IP address **172.17.42.1** is not in use on the host computer, Docker will use it with the psuedo network bridge adapter **docker0**. To enable the Docker containers to use the host computer's DNS provided by dnsmasq, it is adviseable to set the IP address for the pseudo network adapter **docker0** to **172.17.42.1** and also set the dnsmasq DNS server to respond to requests on this ip address. More details below in the installation section on configuring this. If you decide to use another ip address you wil need to change this in a number of files.

**Note:** For Ubuntu 15.04 the place to configuration for Docker has changed. See [Setting Docker’s DOCKER_OPTS on Ubuntu 15.04](http://blog.benhall.me.uk/2015/07/setting-dockers-docker_opts-on-ubuntu-15-04/)


##Network Manager controled dnsmasq
#### Using a dnsmasq.conf configuration file
Using information from a comments that  [Bozzie4](https://gist.github.com/Bozzie4)  and [harish2704](https://gist.github.com/harish2704) made to [magnetikonline](https://gist.github.com/magnetikonline)'s gist. I removed all the options that I could and put them in configuration file (/etc/NetworkManager/dnsmasq.d/dnsmasq.conf) to allow me to override the hardcoded Network Manager dnsmasq configuration.

#### The Issue
I have not found a way to override the hardcoded **--listen-address=127.0.1.1** that Network Manager sends to dnsmasq. This means that all other IP addresses that you want dnsmasq to listen on must be specifically given in the configuration.  I found that if I gave the **docker0** interface address of **172.17.42.1** as an address for dnsmasq to listen on Network Manager would start at boot but that it would fail to start dnsmasq. This is due to the fact that the default upstart job (/etc/init/docker) for docker is waiting for a network interface other than the loopback (net-device-up IFACE**!**=lo) to be up and running before it starts. I removed the exlamation point (!)(which means NOT) to have Docker start once the loopback interface has started. After this Docker would start before Network Manager and because the **docker0** interface had started with the ip address of ** 172.17.42.1** it would also sucessfully start dnsmasq.

#### Running Docker with only the loopback network interface (lo)
When I finally got everthing working with the dns via the Network Manager dnsmasq and docker-gen, I found that I could still not do web development on the train to work as dns did not work without a internet connection. Running `cat /etc/resolv.conf` with the loopback interface (lo)  and the **docker0** up revealed that there were no DNS servers configured to resolve requests. I found that I could assign a DNS server to the loopback interface in /etc/network/interfaces (dns-nameservers 127.0.1.1), but that I had to allow Network Manager to manage network devices in this file to have it change the DNS server on a network change to only the loopback and **docker** interfaces. Allowing Network Manager to manage the network interfaces is done in/etc/NetworkManager/NetworkManager.conf in the **[ifupdown]** section by setting  managed=**true**.

#### The Alternative Method
Using the alternative wrapper method it is possible to use a configuration without any **--listen-address=** line and have dnsmasq listen on all IP adresses. However you must stop Network Manager from running dnsmasq and setup a seperate dnsmasq.  I have added a folder [**dnsmasq_wrapper_alternative**](dnsmasq-configuration/dnsmasq_wrapper_alternative) with this info. See the [**dnsmasq_with_network-manager_for_docker.md**](dnsmasq-configuration/dnsmasq_wrapper_alternative/dnsmasq_wrapper_alternative.mddnsmasq_with_network-manager_for_docker.md) file for more info.


## docker-gen DNS .tmpl  files
#### DNS for .docker
The **dockerhostdns.tmpl** in the **templates** folder will configure the almost default ip address of 172.17.42.1 as the destination for all **.docker** hostnames.

#### DNS for containers
The **dnsmasq.tmpl** in the **templates** folder will create a dns host file for dnsmasq.

## docker-compose.yml
#### Volumes to mount

- `"/var/run/docker.sock:/tmp/docker.sock"`
- `"~/units/dockergen/:/etc/docker-gen/templates"` **- docker-gen templates**
- `"./docker-gen.conf:/etc/docker-gen/conf.d/docker-gen.conf"` **- docker-gen configuration file**
- `"/var/tmp/dockerhosts:/var/tmp/dockerhosts"` **- where the dnsmasq hostname files will be created**
```yaml
volumes:
    - "/var/run/docker.sock:/tmp/docker.sock"
    - "~/units/dockergen/:/etc/docker-gen/templates"
    - "./docker-gen.conf:/etc/docker-gen/conf.d/docker-gen.conf"
    - "/var/tmp/dockerhosts:/var/tmp/dockerhosts"
```

#### Command to run  docker-gen
We can override the run command for the docker-gen container by adding this line to our docker-compose.yml. Her we are telling the container to use our configuration file that we mounted above.
```yaml
command: -config /etc/docker-gen/conf.d/docker-gen.conf
```

#### More info on configuring your docker-compose.yml
See the [docker-compose-localdevmeos](https://github.com/meosch/docker-compose-localdevmeos) project for more info. Specifically the **dockergen:** container **volumes:**, **command:** sections of the **docker-compose_all_containers.yml** file.
Also see the **docker-gen.conf** file from the same project for an example of a docker-gen configuration file.

## Installation
### Network Manager
Change **managed=** to **true** in the **[ifupdown]** section of  [/etc/NetworkManager/NetworkManager.conf](/etc/NetworkManager/NetworkManager.conf) to allow Network Manager to manage the interfaces in [/etc/network/interfaces](file:///etc/network/interfaces)

```bash
[ifupdown]
managed=true
```
Add Network Manager dnsmasq DNS server to the loopback interface. In [/etc/network/interfaces](file:///etc/network/interfaces) add `dns-nameservers 127.0.1.1` under the **lo** network interface. My loopback interface configuration looks like this:
```bash
auto lo
iface lo inet loopback
      dns-nameservers 127.0.1.1
```

### dnsmasq
* Copy the **dnsmasq.conf** in this folder to [/etc/NetworkManager/dnsmasq.d/dnsmasq.conf](file:///etc/NetworkManager/dnsmasq.d/dnsmasq.conf) (from this folder run) 

```bash
cp dnsmasq.conf /etc/NetworkManager/dnsmasq.d/dnsmasq.conf
```

* Create links to the dnsmasq host files that docker-gen will create with:

```bash
sudo ln -s -T /var/tmp/dockerhosts/dockerhosts /etc/NetworkManager/dnsmasq.d/dockerhosts
sudo ln -s -T /var/tmp/dockerhosts/docker /etc/NetworkManager/dnsmasq.d/docker
```

* Create the directory and empty files for dnsmasq to use for configuration when docker-gen has not yet mount. Otherwise dnsmasq will complain that the configuration files do not exist, not start and break your networking.
```bash
mkdir /var/tmp/dockerhosts
touch /var/tmp/dockerhosts/docker
touch /var/tmp/dockerhosts/dockerhosts
```

* Restart Network Manager with 
```bash
sudo restart network-manager
```

### docker
* Copy **docker** to **/etc/default/docker** (if you don't have this file) or add these two lines to your current **/etc/default/docker** file:

```bash
# Always use the same ip address and also use it for dns
DOCKER_OPTS="--bip=172.17.42.1/24 --dns=172.17.42.1"
```
This sets the ip addres for the pseudo network interface **docker0** and also set it as the dns server for the containers.

* As the super user edit [/etc/init/docker.conf](file:///etc/init/docker.conf) to have Docker start when the loopback interface (**lo**) is up. Remove the exclamation point(**!**) from line 3.

<pre><code>start on (local-filesystems and net-device-up IFACE<mark><b>!</mark></b>=lo)
</code></pre>

to
```bash
start on (local-filesystems and net-device-up IFACE=lo)
```
> **Note:** Docker, Upstart **start on** and file systems
 I just ran updates and saw that the Docker Upstart script ([/etc/init/docker.conf](file:///etc/init/docker.conf)) changed the **start on** line. The default line is now:
```bash
start on (filesystem and net-device-up IFACE!=lo)
```
>For the current setup I am proposing with docker-gen and dnsmasq that allows you to work with only the loopback interface I would recommend using **local-filesystems**  instead of **filesystems** which would include network file systems as this will also delay docker from starting and cause dnsmasq startup to fail.

* Restart Docker with
```bash
sudo restart docker
```

## Accessing new containers
As you bring up new containers or restart them you will need to get dnsmasq to reload the docker container DNS files that were created by docker-gen. You can do this by restarting Network Manager with
```bash
sudo restart network-manager
```

## Docker Container access from other computers
If you need to access to your docker containers from other computers you will need use the dnsmasq wrapper method. See **dnsmasq_wrapper_alternative.md**  in the **dnsmasq_wrapper_alternative** folder.

## Todo
* Currently if you do not have a network connection via Network Manager  that is active dnsmasq does not work or return an answer. Find a way to setup a dummy network adapter that is managed by Network Manger to use in this cases.

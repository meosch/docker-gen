# dnsmasq  for Docker containers - Wrapper Script Method
## Background
I trying to get this dnsmasq to work with docker I tried several methods.  This one is based on the one found here: [Using Dnsmasq with Ubuntu for VM web application testing](https://gist.github.com/magnetikonline/6236150).
## Previous Work
[magnetikonline](https://gist.github.com/magnetikonline) used a wrapper script to parse out options sent to dnsmasq.
## Using a configuration file
I took this a step further using information from a comments that  [Bozzie4](https://gist.github.com/Bozzie4)  and [harish2704](https://gist.github.com/harish2704) made to [magnetikonline](https://gist.github.com/magnetikonline) gist. I removed all the options that I could and put them in configuration file to allow me to easier alter the configuration.
## The Problem
The problem I found with using a wrapper script is that if Network Manager was **restarted**  to get new dns entries from new containers that dnsmasq was not restarted and using **stop** with Network Manager only stopped Network Manager and not dnsmasq.bin (what the dnsmasq binary was renamed).
## The Issue
I include this here as there is no way I have found to override the hardcoded **--listen-address=127.0.1.1** that Network Manager sends to dnsmasq. This means that all other IP addresses that you want dnsmasq to listen on must be specifically given in the configuration. Using the wrapper it is possible to use a configuration without any **--listen-address=** line and have dnsmasq listen on all IP adresses. 
## Why include this alternative
This would be usefull if you want other computers to be able to access the containers on your computer. However you would need to setup a DNS for the .docker domain that pointed to your computer that the other computers use for DNS.

## Installation
* Rename the **/usr/sbin/dnsmasq** binary to **/usr/sbin/dnsmasq.bin** ``sudo mv /usr/sbin/dnsmasq /usr/sbin/dnsmasq.bin``
* Copy the **dnsmasq** script in this folder to **/usr/sbin/dnsmasq** (from this folder run) ``sudo cp dnsmasq /usr/sbin/dnsmasq``
* Make **/usr/sbin/dnsmasq** executable ``sudo chmod a+x /usr/sbin/dnsmasq``
* Copy the **dnsmasq.conf** in this folder to **/etc/NetworkManager/dnsmasq.d/dnsmasq.conf** (from this folder run) ``cp dnsmasq.conf /etc/NetworkManager/dnsmasq.d/dnsmasq.conf``
* Restart Network Manager with ``sudo restart network-manager``
## Recommendation
If you do not need access to your docker containers from other computers use the configuration file only method.

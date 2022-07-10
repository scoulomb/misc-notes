
# Setup a lab environment with kubernetes

Objective would be to do technical exploration and watch.
For example to explore tools like ArgoCD:
- https://argo-cd.readthedocs.io/en/stable/
- https://www.youtube.com/watch?v=MeU5_k9ssrs

So stack is VM/Bare metal + Docker (+ Compose) + A kubernetes distribution + Tools (ArgoCD...).

## Setup physical (old) machine with Ubuntu and ssh access

Bare metal + Docker + A kubernetes distribution + Tools (ArgoCD...).

### Create a Ubuntu bootable key with Rufus

- https://releases.ubuntu.com/22.04/
- https://ubuntu.com/tutorials/create-a-usb-stick-on-windows#1-overview


### Setup Docker


````
sudo apt-get update
sudo apt install docker.io
sudo apt install curl
````

### Setup Minikube

https://kubernetes.io/fr/docs/tasks/tools/install-minikube/

````
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
  
sudo mkdir -p /usr/local/bin/
sudo install minikube /usr/local/bin/

sudo usermod -aG docker $USER && newgrp docker # to use Docker driver and not none (VM)
minikube start --driver=docker
````


if kubectl not found do

````
minikube kubectl -- get pods -A
alias k='minikube kubectl'
````

### Add SSH 

#### Setup 

https://www.cyberciti.biz/faq/ubuntu-linux-install-openssh-server/

````
sudo apt-get install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
````

#### Add a NAT rule to access from internet

For example in SFR v8: http://192.168.1.1/network/nat


````
ssh-hp 	TCP 	Port 	22 	192.168.1.90 	22
````


#### Add a DNS config (for example in domains.google.com) 

https://domains.google.com/registrar/coulombel.net


````
Zone coulombel.net
laptop   IN A 109.29.148.109
local.hp IN A 192.168.1.90
````




Note when testing we may have TTL when record not found

#### Test

````
ssh scoulomb@109.29.148.109 (public IP via NAT)
ssh scoulomb@laptop.coulombel.net (public IP via NAT + DNS; need nat if using pub ip even in local network)
ssh scoulomb@192.168.1.90  (private IP)
ssh scoulomb@local.laptop.coulombel.net (private IP via NAT + DNS)
````

#### Avoid credentials in ssh and do not use sshpass

See https://serverfault.com/questions/241588/how-to-automate-ssh-login-with-password

##### I tried where client == server first


Generate a pub/priv key pair for client 
````
ssh-keygen -t rsa -b 2048
cat /home/scoulomb/.ssh/id_rsa.pub
````

Copy client public key to remote server

````
ssh-copy-id scoulomb@laptop.coulombel.net
<input pwd>
````

Check server contain public key 

````
cat .ssh/authorized_keys
````


Log without credentials

````
ssh scoulomb@laptop.coulombel.net
````

##### Same where client !=server

Note if same IP assigned to machine but change of server signature meanwhile do some clean-up

See https://stackabuse.com/how-to-fix-warning-remote-host-identification-has-changed-on-mac-and-linux/

````
$  ssh-keygen -R laptop.coulombel.net
also otherwise offending key
$ ssh-keygen -R 109.29.148.109
````

This clean the client known host file which contains public key of server

Then same process.

Do not define a passphrase as otherwise will need to add it everytime we ssh to remote.


````
<user>@<client-machine> MINGW64 ~
$ ssh-keygen -t rsa -b 2048
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/<user>/.ssh/id_rsa):
/c/Users/<user>/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/<user>/.ssh/id_rsa
Your public key has been saved in /c/Users/<user>/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:vD3rSw3y7rIW9stcBg5EQmlQkg8VFIW9sodF8vSsrVI <user>@<client-machine>
The key's randomart image is:
+---[RSA 2048]----+
|    oBOBo        |
|    o.=o+        |
|     + =.+       |
|      oo+ o      |
|       =Soo      |
|      o EB.+     |
|       +.+* +    |
|      . +=.=     |
|       o.=Xo     |
+----[SHA256]-----+


<user>@<client-machine> MINGW64 ~
$ ls /c/Users/<user>/.ssh
id_rsa  id_rsa.pub  known_hosts  known_hosts.old

<user>@<client-machine> MINGW64 ~
$ cat /c/Users/<user>/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC+KkKw0BfpzBos77xEfNP9BGP2akh2MNlfWE9Yt+rHrnt3aZsvVfHUPCczDSMG/P6XFRhVLIHWaUpHOOVzpzDNzqPcZJjAoYPSfnx/7cd86oI1ZAng0y47IuqvCOVtexvO7+E4KZis+8g50WKc5ITEcDtDIRgK3847UoZnLIUdu7q8B/zuH417TruLxiypDhBAfLzNtQ0QXJeRzzl7oKvUEsGD5nEJyCtmJiWnSjGH09sE8VDKziz9IsEznBocjM0ElgNWxGrxb9yU6yJdlL5YpKcnU9mowSkTP4DArumgt/aSTMVW5anKsJx+gw2DPTWCBeek9bIt4AE6oMVYtABx <user>@<client-machine>

<user>@<client-machine> MINGW64 ~
$ ssh-copy-id scoulomb@laptop.coulombel.net
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/c/Users/<user>/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
scoulomb@laptop.coulombel.net's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'scoulomb@laptop.coulombel.net'"
and check to make sure that only the key(s) you wanted were added.

# check client pub key in authorized !
<user>@<client-machine> MINGW64 ~
$ ssh scoulomb@laptop.coulombel.net
Welcome to Ubuntu 21.10 (GNU/Linux 5.13.0-40-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

186 updates can be applied immediately.
125 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Last login: Fri Apr 22 16:18:15 2022 from 192.168.1.1
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat .ssh/authorized_keys
[...]
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC+KkKw0BfpzBos77xEfNP9BGP2akh2MNlfWE9Yt+rHrnt3aZsvVfHUPCczDSMG/P6XFRhVLIHWaUpHOOVzpzDNzqPcZJjAoYPSfnx/7cd86oI1ZAng0y47IuqvCOVtexvO7+E4KZis+8g50WKc5ITEcDtDIRgK3847UoZnLIUdu7q8B/zuH417TruLxiypDhBAfLzNtQ0QXJeRzzl7oKvUEsGD5nEJyCtmJiWnSjGH09sE8VDKziz9IsEznBocjM0ElgNWxGrxb9yU6yJdlL5YpKcnU9mowSkTP4DArumgt/aSTMVW5anKsJx+gw2DPTWCBeek9bIt4AE6oMVYtABx <user>@<client-machine>

````
### known hosts

http://www.octetmalin.net/linux/tutoriels/ssh-fichier-known_hosts.php


> Le fichier "known_hosts" ce trouve dans un dossier caché a la racine du répertoire de l'utilisateur.
"~/.ssh/known_hosts" ou le chemin en entier "/home/[nom_utilisateur]/.ssh/known_hosts".

> Ce fichier contient la clé publique de tous les serveurs ssh sur lequelle ce compte c'est connecté.
Ces clés "dsa" publique proviennent du fichier "/etc/ssh/ssh_host_dsa_key.pub" qui ce trouve sur le serveur distant.

known_host stores server's public key on client side.
The public key of the hostself is stored there: /etc/ssh/ssh_host_ecdsa_key.pub in the server

Example where client = server

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ ls /etc/ssh/
moduli      ssh_config.d  sshd_config.d       ssh_host_ecdsa_key.pub  ssh_host_ed25519_key.pub  ssh_host_rsa_key.pub
ssh_config  sshd_config   ssh_host_ecdsa_key  ssh_host_ed25519_key    ssh_host_rsa_key          ssh_import_id
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /etc/ssh/ssh_host_ecdsa_key.pub
ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBAGKn4yeRrczgFduirAP27KjTCMNrsGzbc6Qe9wuCJiV0b0jFPBowyB20J41GZgCW+G/kewLux5v6nNpO+pzjU= root@scoulomb-HP-Pavilion-TS-Sleekbook-14
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat .ssh/known_hosts
|1|CMmw3968zRowaUx8mjXX39s1HrU=|wNZMtoRKSYzHDeaT/u2IAb8/1jg= ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBAGKn4yeRrczgFduirAP27KjTCMNrsGzbc6Qe9wuCJiV0b0jFPBowyB20J41GZgCW+G/kewLux5v6nNpO+pzjU=
|1|+qsCL1HnJxZozkc0AbuEs1ylUrA=|S808vlm2QosUfxt9KiZwue8X8A8= ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBAGKn4yeRrczgFduirAP27KjTCMNrsGzbc6Qe9wuCJiV0b0jFPBowyB20J41GZgCW+G/kewLux5v6nNpO+pzjU=
````

### SSH Summary

https://serverfault.com/questions/935666/ssh-authentication-sequence-and-key-files-explain

> quoting answer

The following answer explains the files needed to prepare for ssh authentication using public-private key pairs ("Public Key Infrastructure" or "PKI"), and how those files are used during an actual ssh session. Some specifics here use names and directories that apply to linux, but the principles apply across all platforms, which use programs and files parallel to these. The main features of interest are:

 - **User machine**
   - **User key pair:** Public and private, which the client-side user has to create using ssh-keygen, creating files with names such as:
     - ~/.ssh/**id_rsa**    (private)  and 
     - ~/.ssh/**id_rsa.pub**    (public)
     - In preparation, must be given to host's **authorized_keys** file
   - ~/.ssh/**known_hosts**
     - which receives the public key from the server, if user accepts it on first log in.
 - **Host (server) machine**
   - **Host key pair:** Public and Private
     - are automatically created at some point like installation of openssh on the server. Typical names:
       - /etc/ssh/**ssh_host_rsa_key**      (private)
       - /etc/ssh/**ssh_host_rsa_key.pub**    (public)
     - host offers the public key to the client-side user the first time the client-side user tries to connect with ssh. Client will store host's key in **known_hosts**
   - ~/.ssh/**authorized_keys**
     - In preparation, must be given the public key of each user who will log in.
 - **Sequence of events** in an actual SSH (or rsync) session, showing how the files are involved.

(Note that there are two different common signature algorithms, RSA and DSA, so where this discussion uses 'rsa', the string 'dsa' could appear instead.)


**Configuration/preparation**
[![enter image description here][1]][1]

**SSH connection and use**
[![enter image description here][2]][2]

  [1]: ./media/pC0Qn.png
  [2]: ./media/4cZbh.png

### dyndns

In previous section laptop.coulombel.net points to box ip but IP could change.
It could be better to use dyndns.

Google domains supports dyndns:

https://support.google.com/domains/answer/6147083?authuser=1&hl=en#zippy=%2Cset-up-a-client-program-on-your-gateway-host-or-server%2Cuse-the-api-to-update-your-dynamic-dns-record

We will install ddclient

````
sudo apt install ddclient
sudo dpkg-reconfigure ddclient
cat vi /etc/ddclient.conf
sudo ddclient -daemon=0 -debug -verbose -noquiet
# add -force option if no update done (https://sourceforge.net/p/ddclient/wiki/usage/)
sudo ddclient -daemon=0 -debug -verbose -noquiet -force
````

We can now see nslookup and google domain UI return box IP.

Note ddclient offers 2 options

````
The method ddclient uses to determine your current IP address. 
Your  options:             

Web-based IP discovery service: Periodically visit a web page that shows your IP address. You probably want this option if your computer is connected to the Internet via a Network Address    
Translation (NAT) device such as a typical consumer router.                                                                                                                                                                                                                                                             Network interface: Use the IP address assigned to your computer's network interface (such as an Ethernet adapter or PPP connection). You probably want this option if your computer connects   directly to the Internet (your connection does not go through a NAT device)                                                                                                         
                                    
````


If we use network interface and doing NAT, we suppose it will return private IP. Let's try:


````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ ip a 
[...]
3: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 20:68:9d:c5:99:c1 brd ff:ff:ff:ff:ff:ff
    altname wlp1s0
    inet 192.168.1.90/24 brd 192.168.1.255 scope global dynamic noprefixroute wlo1
````

We will use wlo1 interface, we expect IP to be 192.168.1.90.
After update we can see nslookup and google domain UI will return priv ip 192.168.1.90


Google has actually API to setup DNS (used by ddns).

We could setup dyndns on the box directly (equivalent to use network interface mode without NAT).
Unfortunately SFR Box v8 does not support google as dyndns provider.
It supports noip service (https://www.noip.com/). So we could use noip and then have a CNAME, to no IP A dynamic record.
no-ip is also useful when dns provider does ntot support dynamic dns (such as Gandi).
noip is however painful as not free and need to renew domain regulalrly.
noip as the DUC client equiavlent to DDNS.

A solution could be to use qnap dyndns service which come for free (scoulomb.myqnapcloud.com). Then we have CNAME recrod to scoulomb.myqnapcloud.com dynamic  A record.
Advantage is that NAS always plugged so provide IP to box. Alternative solution could be to have ddns deployed as docker in NAS (not tested).


In local network we can define private DNS: 
http://192.168.1.1/network/dns

## Other methods

See [other methods](./others.md)

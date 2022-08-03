# Jupter setup - appendix

## Context

Idea is to setup Jupyter on top of [Lab env](../lab-env/README.md).

Why?
- Exploration notes
- And colloaboration at 2 with screen sharing 

<!-- alternative to use MS teams in web-->

## Setup

### Local

From https://jupyter.org/install

````
pip install jupyterlab
````

Then add bash kernel 

````
https://pypi.org/project/bash_kernel/
pip install bash_kernel
python3 -m bash_kernel.install
````


Trigger Jupyer and go to 

`127.0.0.1:8888` or `localhost:8888`.

### LAN

If you want a friend to access from another machine in the LAN do (use git bash, wsl (console only), powershell, windows terminal... will also work)

<!-- disable any VPN -->

````
$ ssh scoulomb@192.168.1.90
[....]
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ jupyter-lab --ip 192.168.1.90 --port 8080
[....]
[C 2022-08-03 18:09:43.709 ServerApp]

    To access the server, open this file in a browser:
        file:///home/scoulomb/.local/share/jupyter/runtime/jpserver-9573-open.html
    Or copy and paste one of these URLs:
        http://192.168.1.90:8080/lab?token=69784938cfe87b6a458372f0671f100c879d5a847d321d74
     or http://127.0.0.1:8080/lab?token=69784938cfe87b6a458372f0671f100c879d5a847d321d74
````


But not access via `127.0.0.1`, `localhost` will not work from machine where Jupyter is deployed. We have to use `192.168.1.90`. Token is painful to copy
So listen on all interface and specify the token.

````
jupyter-lab --ip 0.0.0.0 --port 8080 --NotebookApp.token='hello'
````

We also explicily defined the port.

Om machine where Jupyer is deployed go to  `127.0.0.1:8080`, `localhost:8080`.
From machine in LAN go to `192.168.1.90:8080`.


### WAN - Allow remote access via [NAT in the BOX](https://github.com/scoulomb/docker-under-the-hood/blob/main/NAT.md)
<!-- bit of inception here ;) -->

Conigure NAT rule at http://192.168.1.1/network/nat in SFR box.
````
jupyter 	TCP 	Port 	8081 	192.168.1.90 	8080
````    

<!-- a port > 10000 can be blocked by browser -->

And access from internet to 

`http://109.29.148.109:8081/` or 

Given DNS setup made [here](../lab-env/README.md#dyndns) we can access via `http://home.coulombel.net:8081`.


This is accessible from any phone in 4G.
<!-- and corp laptop on VPN (unlike private IP) -->

Combined with [Wake On WAN/LAN](../NAS-setup/Wake-On-LAN.md) applied to laptop could be promising (required wired).

<!--J: note we could dockerize Jupyter (but linux bash is the one from container), setup oc client to have a cool way to access openshift client
We could deploy this container inside the PAAS and OC client car target control plane from inside the PaaS in some cases
This would avoid going through jumpsrv, and requires container to access  DNS of the PAAS (corp DNS in resolv.conf)  -->

<!-- [Here we had setup Jupyer](../lab-env/others.md#use-archlinuxubuntu) -->
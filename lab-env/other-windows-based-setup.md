# Kubernetes on Windows based stack

In those stack we setup Kubernetes on Windows without **Linux** or a **standard Linux VM**.
Note that WSL 2 is not a standard linux VM.

From https://docs.microsoft.com/en-us/windows/wsl/compare-versions
> WSL 2 uses the latest and greatest in virtualization technology to run a Linux kernel inside of a lightweight utility virtual machine (VM). However, WSL 2 is not a traditional VM experience.

This requires a good understanding of [Kubernetes distribution](./kubernetes-distribution.md).

## WSL and Docker for Windows and embeeded k8s

> `Windows Host  + Hyper-v + WSL2 + [Docker Desktop for windows] = { Docker (+ Compose) + https://docs.docker.com/desktop/windows/#kubernetes} + Tools (ArgoCD...).`

<!-- 
Already used this for git secret project and faced dns issue here corp solved with 8.8.8.8 DNS: https://github.com/scoulomb/misc-notes/blob/master/github-security/README.md -->

### Install WSL

Install WSL: https://docs.microsoft.com/en-us/windows/wsl/install
https://web.archive.org/web/20220716071031/https://docs.microsoft.com/en-us/windows/wsl/install

I had some issue here, with an old wsl version

````
PS C:\Windows\system32> wsl --install -d Ubuntu
No action was taken as a system reboot is required.
````

I fixed it by going to manual page to install linux kernel update
https://docs.microsoft.com/en-us/windows/wsl/install-manual#step-4---download-the-linux-kernel-update-package

But most importantly I restart windows with update mode

It triggered Ubuntu window and it is working :)

Proof: can target lab laptop from WSL
<!-- same from work laptop -->

````
scoulomb-wsl@DESKTOP-5NN6O2B:~$ ssh scoulomb@192.168.1.90
scoulomb@192.168.1.90's password:
Welcome to Ubuntu 21.10 (GNU/Linux 5.13.0-52-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
````

Note we have 
- cmd
- Powershell
- Can start vm from program 
- Windows terminal: https://docs.microsoft.com/fr-fr/windows/wsl/install#ways-to-run-multiple-linux-distributions-with-wsl (from windows 11)

### File sharing in WSL


Share file: \\wsl$\Ubuntu\home\scoulomb-wsl
As described here https://dev.to/miftahafina/accessing-wsl2-files-from-windows-file-explorer-308o (access wsl folder from windows, requires network which may be locked in corporate envrionmet)

From WSL can also do `cd /mnt/c`.

Can also land in current path by lanching `wsl` from powershell/cmd.
<!-- manage to write file from here on corp laptop, in my personal laptop seem permission issue -->

<!--
This seems not available anymore? https://devblogs.microsoft.com/commandline/access-linux-filesystems-in-windows-and-wsl-2/ (mount windows folder to wsl)
-->


### Docker Desktop for windows

Install Docker Desktop for windows: https://docs.docker.com/desktop/install/windows-install/
https://web.archive.org/web/20220716071031/https://docs.docker.com/desktop/install/windows-install/

Tick use WSL-2 instead of Hyper-V

Note links with certificate: https://docs.docker.com/desktop/windows/#adding-tls-certificates

In settings > Resource > WSL integration
Ensure we have 
- Enable integration with my default WSL distro
- Enable integration with additional distros

#### Basic demo

We can trigger Docker from Windows.
See demo when using git docker image to clone a repo from which they build an image!

````
docker run --name repo alpine/git clone https://github.com/docker/getting-started.git
docker cp repo:/git/getting-started/ .'
````

We can access to both docker and docker-compose on windows + wsl :)

Note windows/wsl accept `docker compose` and `docker-compose` command.
When we do `docker-compose` from windows it actually creates it in WSL.
Can be easily checked via `docker ps` in WSL.

#### Advanced demo


Assume we setup home assistant via Docker-compose file: https://github.com/scoulomb/home-assistant
I could not access UI.
We could think it is a port forwarding issue. Tried to fix with workaround here (but actually more for public access):
- https://gist.github.com/stormwild/f464fae904212d4334b3905655a2218c,
- https://www.williamjbowman.com/blog/2020/04/25/running-a-public-server-from-wsl-2/ 
- https://web.archive.org/web/2021*/https://www.williamjbowman.com/blog/2020/04/25/running-a-public-server-from-wsl-2/
But internal curl did not work too!

It is a problem due to host network!
From https://docs.docker.com/network/network-tutorial-host/#prerequisites
> The host networking driver only works on Linux hosts, and is not supported on Docker Desktop for Mac, Docker Desktop for Windows, or Docker EE for Windows Server.

See https://community.home-assistant.io/t/homeassistant-running-on-docker-desktop-for-windows-but-not-accessible-from-browser/370731

If we deploy https://github.com/open-denon-heos/remote-control/blob/main/PRD.docker-compose.yaml we can access UI via port 8000 (as container do forward) but not 5000 (host network).
Not we did not have to configure firewall rule or port forwarding between WSL and windows.



### Hyper-v ? What is that?

#### Hyper-V and WSL 2

https://docs.microsoft.com/fr-fr/virtualization/hyper-v-on-windows/about/
In setup page we make the distinction between WSL backend and Hyper-V and windows container (https://docs.docker.com/desktop/install/windows-install/)...

Best question answer/here <3

Question
1. Is Hyper-V required or not for WSL 2?
2. Can Hyper-V be activated or not on Windows 10 Home

Answer
1. No, for WSL2 you only need "Virtual Machine Platform" (a subset of Hyper-V) and "Windows Subsystem for Linux" features, both available in Home edition.
2.    No. Full Hyper-V is not available in Home.

"Please enable the Virtual Machine Platform windows feature and ensure virtualization is enabled in the BIOS" 

Consistent with manual installation procedure of WSL 2: https://docs.microsoft.com/en-us/windows/wsl/install-manual 
<!-- had followed that one in corp env -->

````
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
````

Also note from https://docs.microsoft.com/en-us/windows/wsl/compare-versions
> WSL 2 uses the latest and greatest in virtualization technology to run a Linux kernel inside of a lightweight utility virtual machine (VM). However, WSL 2 is not a traditional VM experience.


#### Conflict Hyper-V (needed for WSL, and docker for windows needs WSL ) and virtual box

See [other/vagrant](others.md#use-vagrant-vm)/

Note that docker for Windows needs hyper V feature so it can not work at the same time as Vagrant: https://docs.microsoft.com/en-us/troubleshoot/windows-client/application-management/virtualization-apps-not-work-with-hyper-v, https://docs.docker.com/desktop/windows/install/

We can now use WSL 2 backend and Docker 4 windows which requires wsl 2 features (which uses subset of Hyper v). Is it still mutually exclusive? YES :(
From https://forums.virtualbox.org/viewtopic.php?t=95426
> Yes, WSL2 is not compatible with Virtualbox, due to WSL2 using Hyper-V, which uses VT-x exclusively and doesn't share it with Virtualbox. To use Virtualbox properly, for now*, you have to have Hyper-V off, which turns off anything that uses Hyper-V.
See
- https://4sysops.com/archives/install-windows-subsystem-for-linux-wsl-in-windows-11/#:~:text=While%20WSL%202%20uses%20Microsoft's,works%20perfectly%20fine%20without%20it.
- https://docs.docker.com/desktop/windows/install/

See below [mix wsl with k3s, minikube](#mix-wsl-with-k3s-minikube)


### Kubernetes in Docker for windows


Had issue and followed: https://stackoverflow.com/questions/63312861/docker-desktop-kubernetes-failed-to-start

Cmd (or powershell)
````
C:\Users\sylva>kubectl get nodes
NAME             STATUS   ROLES           AGE     VERSION
docker-desktop   Ready    control-plane   5m53s   v1.24.1
````

And WSL 

````
scoulomb-wsl@DESKTOP-5NN6O2B:~/remote-control$ kubectl get nodes
NAME             STATUS   ROLES           AGE     VERSION
docker-desktop   Ready    control-plane   6m36s   v1.24.1
scoulomb-wsl@DESKTOP-5NN6O2B:~/remote-control$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS     NAMES
29c11deada1f   8c2c38aa676e           "/kube-vpnkit-forwar…"   5 minutes ago   Up 5 minutes             k8s_vpnkit-controller_vpnkit-controller_kube-system_5aae0042-eb63-4770-b199-7356cd51b3a8_0
1c4fad702dfe   99f89471f470           "/storage-provisione…"   5 minutes ago   Up 5 minutes             k8s_storage-provisioner_storage-provisioner_kube-system_abb273dd-2f7a-46b7-bab4-1282e020bddc_0
91383c4c7611   k8s.gcr.io/pause:3.7   "/pause"                 5 minutes ago   Up 5 minutes             k8s_POD_vpnkit-controller_kube-system_5aae0042-eb63-4770-b199-7356cd51b3a8_0
32ed0ccef3ac   k8s.gcr.io/pause:3.7   "/pause"                 5 minutes ago   Up 5 minutes             k8s_POD_storage-provisioner_kube-system_abb273dd-2f7a-46b7-bab4-1282e020bddc_0
eae60ce94454   a4ca41631cc7           "/coredns -conf /etc…"   6 minutes ago   Up 5 minutes             k8s_coredns_coredns-6d4b75cb6d-n2nqp_kube-system_67114205-54d9-4fa3-ab0f-c883e1096a64_0
e8bb99b19d7e   a4ca41631cc7           "/coredns -conf /etc…"   6 minutes ago   Up 6 minutes             k8s_coredns_coredns-6d4b75cb6d-c6rfq_kube-system_1dff9243-fbf8-4f2f-9ec8-750be8e07ec3_0
adbadef5ce52   beb86f5d8e6c           "/usr/local/bin/kube…"   6 minutes ago   Up 6 minutes             k8s_kube-proxy_kube-proxy-rssqm_kube-system_d2ccfc2c-a714-4160-8190-9c6c5f8c0e5e_0
1b41d0662172   k8s.gcr.io/pause:3.7   "/pause"                 6 minutes ago   Up 6 minutes             k8s_POD_coredns-6d4b75cb6d-n2nqp_kube-system_67114205-54d9-4fa3-ab0f-c883e1096a64_0
f6b1c00e2473   k8s.gcr.io/pause:3.7   "/pause"                 6 minutes ago   Up 6 minutes             k8s_POD_coredns-6d4b75cb6d-c6rfq_kube-system_1dff9243-fbf8-4f2f-9ec8-750be8e07ec3_0
5c489668f09a   k8s.gcr.io/pause:3.7   "/pause"                 6 minutes ago   Up 6 minutes             k8s_POD_kube-proxy-rssqm_kube-system_d2ccfc2c-a714-4160-8190-9c6c5f8c0e5e_0
2da4695dd3a3   b4ea7e648530           "kube-controller-man…"   6 minutes ago   Up 6 minutes             k8s_kube-controller-manager_kube-controller-manager-docker-desktop_kube-system_bd266f9c253c9118b45703bf4302488d_1
94574226b41a   aebe758cef4c           "etcd --advertise-cl…"   6 minutes ago   Up 6 minutes             k8s_etcd_etcd-docker-desktop_kube-system_2449ddc0985e3be8dd23ffc4d12cb53b_0
3a1539a80e1f   e9f4b425f919           "kube-apiserver --ad…"   6 minutes ago   Up 6 minutes             k8s_kube-apiserver_kube-apiserver-docker-desktop_kube-system_6f74d5bdfeda1c15af1b3f27e176ae41_0
bea5c3053393   18688a72645c           "kube-scheduler --au…"   6 minutes ago   Up 6 minutes             k8s_kube-scheduler_kube-scheduler-docker-desktop_kube-system_8f6e5e140446545893b1ac5a088630f3_0
8e3201d38365   k8s.gcr.io/pause:3.7   "/pause"                 6 minutes ago   Up 6 minutes             k8s_POD_etcd-docker-desktop_kube-system_2449ddc0985e3be8dd23ffc4d12cb53b_0
7217bf72189a   k8s.gcr.io/pause:3.7   "/pause"                 6 minutes ago   Up 6 minutes             k8s_POD_kube-apiserver-docker-desktop_kube-system_6f74d5bdfeda1c15af1b3f27e176ae41_0
27191cea9c0c   k8s.gcr.io/pause:3.7   "/pause"                 6 minutes ago   Up 6 minutes             k8s_POD_kube-scheduler-docker-desktop_kube-system_8f6e5e140446545893b1ac5a088630f3_0
6eeddc9111f0   k8s.gcr.io/pause:3.7   "/pause"                 6 minutes ago   Up 6 minutes             k8s_POD_kube-controller-manager-docker-desktop_kube-system_bd266f9c253c9118b45703bf4302488d_0
````


If we take deployment from: https://github.com/scoulomb/misc-notes/blob/master/lab-env/kubernetes-distribution.md#multinode-cluster-observations 

````
$ kubectl get po
NAME                    READY   STATUS    RESTARTS   AGE
hello-fb5996785-7lxxt   1/1     Running   0          2m28s
hello-fb5996785-phtkv   0/1     Pending   0          2m28s # due to affinity
````

We do not see container and image in Desktop UI created from there.

Will not do: https://docs.microsoft.com/fr-fr/windows/wsl/tutorials/wsl-containers#develop-in-remote-containers-using-vs-code

Note that https://docs.docker.com/desktop/windows/#kubernetes. is not k3s: https://web.archive.org/web/20210620005051/https://www.grottedubarbu.fr/kubernetes-fast-windows/.

## Minikube on Windows

### Minikube runs in Windows host

> `Windows Host  + Minikube with virtual box driver/hyper-V driver + Tools (ArgoCD...).`

Where Minikube can deploy node using 
- Virtual-Box driver (like KVM2 driver in Ubuntu)
- Hyper-V driver (like kVM2 driver in Ubuntu)

Those 2 driver can not work at same time. Same comment as [mentionned above](#conflict-hyper-v-needed-for-wsl-and-docker-for-windows-needs-wsl--and-virtual-box).

See https://rharshad.com/kubernetes-minikube-windows-setup/#running-kubernetes-cluster-in-default-mode; https://web.archive.org/web/20200603144422/https://rharshad.com/kubernetes-minikube-windows-setup/
See Minikube driver: https://minikube.sigs.k8s.io/docs/drivers/.

Here Hyper-v runs k8s node instead of WSL!

### Minikube runs in WSL

> `Windows Host  + Hyper-v + WSL2 + Minikube with Docker driver + Tools (ArgoCD...).`

Thus `VM + container` and not only `VM` in this page for Docker driver: https://minikube.sigs.k8s.io/docs/drivers/#windows. Dockers requires WSL 2 in windows.
We could use a Linux VM but come back to kubernetes on Linux stack case.

See example here: https://medium.com/@edmond.dantes/setup-minikube-with-docker-driver-2e27dc03f196

Minikube deploy nodes in Docker (and workload is docker in docker)

We described [hyper V abvve](#hyper-v--what-is-that).

or using None driver
> `Windows Host  + Hyper-v + WSL2 + Minikube with None driver (1 Node) + Tools (ArgoCD...).`

See https://github.com/kubernetes/minikube/issues/5392#issuecomment-539289653

See parallel with [Minikube Linux driver](./kubernetes-distribution.md#so-if-we-summarize-minikube-driver-mode-linux)

## k3s on WSL 2

> `Windows Host  + Hyper-v + WSL2 + k3s + Tools (ArgoCD...).`

Here we also show how to have k3s distribution: https://www.grottedubarbu.fr/k3s-on-wsl2/

k3d should be possible.

## Docker destop on Linux comment

Note Docker Destop can now run on Linux but everything is virtualized for consistent expereince accross platform: https://docs.docker.com/desktop/install/linux-install/, https://docs.docker.com/desktop/install/linux-install/#why-docker-desktop-for-linux-runs-a-vm
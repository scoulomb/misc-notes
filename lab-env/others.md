# Other Kubernetes setup on Linux based stack


In [README](./README.md#generic-stack-is) refer to generic Linux stack.


> `Bare metal Linux (or Bare metal + VM Linux) + Docker (+ Compose) + A kubernetes distribution (deployed in VM/Docker) + Tools (ArgoCD...)`.

Note we have seen in [Kubernetes distribution](./kubernetes-distribution.md) section, that Kubernetes distribution itself can deploy VM/Docker/bare metal (single node) for k8s Nodes. 

## Use dual boot on recent machine

> `Bare metal Linux host dual boot (with Windows) + Docker (+ Compose) + [A kubernetes distribution](./kubernetes-distribution.md#so-if-we-summarize-minikube-driver-mode) + Tools (ArgoCD...).`

- Make OS recovery tool: https://www.dell.com/support/home/fr-fr/drivers/osiso/recoverytool
- Boot to usb: https://www.dell.com/support/kbdoc/fr-fr/000126121/acc%C3%A8s-%C3%A0-la-configuration-syst%C3%A8me-uefi-bios-sous-windows-sur-votre-syst%C3%A8me-dell (access option via windows, search uefi and advanced startup)
- But had issue with Intel RST: https://help.ubuntu.com/rst


## Use Vagrant VM 

> `Windows bare metal + (vbox) VM Linux + Docker (+ Compose)  + [A kubernetes distribution](./kubernetes-distribution.md#so-if-we-summarize-minikube-driver-mode) + Tools (ArgoCD...)`.

### Could use Rancher VM

- https://rancher.com/docs/rancher/v2.5/en/quick-start-guide/deployment/quickstart-vagrant/
- VT-X+Enabling can check it is enable via taskmgr/CPU (then docker Destop for windows will not work)
- https://superuser.com/questions/1153470/vt-x-is-not-available-but-is-enabled-in-bios
- Solved vtx issue by doing 

````
dism.exe /Online /Disable-Feature:Microsoft-Hyper-V
````

Managed to setup rancher.
Rancher contains docker but it does not contain kube :(.


### Use Archlinux/Ubuntu 

See https://github.com/scoulomb/myk8s/tree/master/Setup.
In link we were using:
- VM Linux = vbox + kubeadm
- VM Linux = minikube with none driver

In VM linux we could have had VM driver for node (container in VM node in VM) or Docker Driver (container in container in VM)

Compliant with [kubernetes distribution on VM](./kubernetes-distribution.md)

Some tips to generate vagrant file

https://app.vagrantup.com/ubuntu

````
vagrant init ubuntu/focal64 # change ram > 1024 for minikube
vagrant up
````

if issue with ssh use virtual box UI, login with u/p vagrant vagrant

Same procedure for k8s/docker as in [README.md](./README.md) after, but experienced several issues.
We replace host machine by a VM.

## Use NAS

We can setup Kube 

### Ubuntu station 

> `QTS Linux based OS + Linux Ubuntu station VM +  Docker (+ Compose) + A kubernetes distribution + Tools (ArgoCD...)`.

I wanted to use Ubuntu station but error ticket open: https://service.qnap.com/fr-fr/user/support-detail-single/5002s00000LvvLbAAJ#

Ticket solved but consumes a lot of CPU.

### Container station and compose

Note NAS container sttation can deploy Docker image and also support compose, used here: https://github.com/open-denon-heos/remote-control, https://github.com/scoulomb/home-assistant

### Use container station with k3*d* image 


Not sure we can run this image directy. 
https://hub.docker.com/r/rancher/k3d
See [k3d setup](./kubernetes-distribution.md#k3d).


### But discover container station can deploy a k3*s* natively

> `NAS QTS Linux based OS + Docker (+ Compose) + A kubernetes distribution + Tools (ArgoCD...)`.

> Preference > kubernetes

- Then can setup in windows kubectl
https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/

- Download kubeconfig from qnap: https://kubernetes.io/fr/docs/tasks/access-application-cluster/configure-access-multiple-clusters/

and use it

````
kubectl --kubeconfig ./k3s.yaml config view
# https://github.com/scoulomb/myk8s/search?q=kubectl+run+nginx
kubectl --kubeconfig ./k3s.yaml run nginx --image=nginx --restart=Never --port=80 --expose --dry-run=server -o yaml
kubectl --kubeconfig ./k3s.yaml run nginx --image=nginx --restart=Never --port=80 --expose -o yaml
  
  
kubectl --kubeconfig ./k3s.yaml create namespace argocd
kubectl --kubeconfig ./k3s.yaml apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
````

See https://github.com/scoulomb/misc-notes/blob/master/lab-env/others.md#use-container-station-with-k3d-image 
<!-- disconnect corp vpn -->
  

  
then argocd: https://argo-cd.readthedocs.io/en/stable/cli_installation/

if powershell issue use gtibash

````
$ /c/bin/kubectl --kubeconfig ./k3s.yaml patch svc argocd-server -n argocd -p '{"spec":{"type": "LoadBalancer"}}'
service/argocd-server patched
````

unfortuantely can not access UI, portforwarding does not work
https://argo-cd.readthedocs.io/en/stable/getting_started/
even with 
https://www.qnap.com/fr-fr/how-to/tutorial/article/comment-utiliser-browser-station

<!-- I deploy argocd here to show port issue -->

Next: [Other Kubernetes setup on Windows based stack](./other-windows-based-setup.md)
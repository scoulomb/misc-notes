# Other methods

## Use recent machine

- Make OS recovery tool: https://www.dell.com/support/home/fr-fr/drivers/osiso/recoverytool
- Boot to usb: https://www.dell.com/support/kbdoc/fr-fr/000126121/acc%C3%A8s-%C3%A0-la-configuration-syst%C3%A8me-uefi-bios-sous-windows-sur-votre-syst%C3%A8me-dell (access option via windows, search uefi and advanced startup)
- But had issue with Intel RST: https://help.ubuntu.com/rst


## Use Vagrant 

### Could use Rancher

- https://rancher.com/docs/rancher/v2.5/en/quick-start-guide/deployment/quickstart-vagrant/
- VT-X+Enabling can check it is enable via taskmgr/CPU (then docker 4 windows will not work)
- https://superuser.com/questions/1153470/vt-x-is-not-available-but-is-enabled-in-bios
- Solved vtx issue by doing 

````
dism.exe /Online /Disable-Feature:Microsoft-Hyper-V
````

Managed to setup rancher.
Rancher contains docker but it does not contain kube :(.

Note that intitally docker 4 windows needs hyper V feature so it can not work at the same time as Vagrant: https://docs.microsoft.com/en-us/troubleshoot/windows-client/application-management/virtualization-apps-not-work-with-hyper-v, https://docs.docker.com/desktop/windows/install/

But we can now use WSL 2 backend for docker 4 windows which requires wsl 2 features (which uses Hyper v) so still mutually exclusive
- https://forums.virtualbox.org/viewtopic.php?t=95426
> Yes, WSL2 is not compatible with Virtualbox, due to WSL2 using Hyper-V, which uses VT-x exclusively and doesn't share it with Virtualbox. To use Virtualbox properly, for now*, you have to have Hyper-V off, which turns off anything that uses Hyper-V.
- https://4sysops.com/archives/install-windows-subsystem-for-linux-wsl-in-windows-11/#:~:text=While%20WSL%202%20uses%20Microsoft's,works%20perfectly%20fine%20without%20it.
- https://docs.docker.com/desktop/windows/install/



### Use Archlinux/Ubuntu 

See https://github.com/scoulomb/myk8s/tree/master/Setup

Some tips to gnerate vagrant file

https://app.vagrantup.com/ubuntu

````
vagrant init ubuntu/focal64 # change ram > 1024 for minikube
vagrant up
````

if issue with ssh use virtual box UI, login with u/p vagrant vagrant

Same procedure for k8s/docker as [README.md](./README.md) after, but experienced several issues.


## Use NAS

We can setup Kube 

### Ubuntu station 

I wanted to use Ubuntu station but error ticket open: https://service.qnap.com/fr-fr/user/support-detail-single/5002s00000LvvLbAAJ#

Ticket solved but consumes a lot of CPU.

### Use container station with k3*d* image 

https://hub.docker.com/r/rancher/k3d

To run the container set prvilege option.
> Create > advanced settings > device > run container in privileged mode 

### But discover contaner station can deploy a k3*s* natively

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


### Use WSL and k3s

Not tried

<!-- some dns issue here corp -->

# Kubernetes distribution

Alternative Kubernetes distribution in this stack:

> `Bare metal Linux + Docker + A kubernetes distribution + Tools (ArgoCD...)`.

## Minikube, Docker in Docker and local multicluster 

### Drivers

Note on minikube driver.
Minikube support several driver. For example in Linux:
https://minikube.sigs.k8s.io/docs/drivers/#linux


- Docker - container-based (preferred)
- KVM2 - VM-based (preferred)
- VirtualBox - VM
- QEMU - VM (experimental)
- None - bare-metal
- Podman - container (experimental)
- SSH - remote ssh


In our [setup](./README.md) we propose to use docker driver.
- In that case Docker container (container in pod) will actually be deployed in Docker container (node emulation) in the `host`,
- A Kubernetes node is actually a Docker container, and container pod runs inside the container node (Docker in Docker).
- If we use VM driver, kubernetes nodes will be virtual machine in the host.
- If we use None driver (bare metal), we will have the only option to deploy a single node directly on the machine.
- SSH option enable to have node deployed in remote machine/VM.

In our setup `host` is bare metal but it could be itself a VM.

<!-- kuberntes process can be itself deployed in node (bare-metail in Node) or in Docker,
see ETCD: https://etcd.io/docs/v3.4/op-guide/container/#bare-metal and [here](#comment-on-master-nodes) case of deployment in Docker-->

Here is an example with None driver (bare metal) deployed on a VM: https://github.com/scoulomb/myk8s/tree/master/Setup/MinikubeSetup, [other.md](others.md#use-archlinuxubuntu).

### So if we summarize minikube driver mode (Linux)

- [1] Docker - container-based (preferred): So Docker container runs in Node which are itself container in host machine (which can be (VirtualBox) VM/bare metal)
- [2] VirtualBox/KVM2 - VM: So Docker container runs in Node which are itsel VirtualBox VM in host machine (which can be (VirtualBox) VM/bare metal)
- [3] None - bare-metal: So Docker container runs in Node which is the host machine (which can be VM/bare metal)

We will see that [1] <=> k3d, [3] <=> k3s, kubeadm


### Multinode cluster setup


Thus Docker driver enables to deploy locally a multcluster node unlike None/bare metal driver:

Multinode is failing with None driver

````
$ sudo minikube start --nodes 2 -p multinode-demo --driver=none
[sudo] password for scoulomb:
😄  [multinode-demo] minikube v1.25.2 on Ubuntu 21.10
✨  Using the none driver based on user configuration

❌  Exiting due to DRV_UNSUPPORTED_PROFILE: The 'none driver does not support multiple profiles: 
````

use Docker driver as explained here: https://minikube.sigs.k8s.io/docs/tutorials/multi_node/

````
kubeclt 
`````

We can see 2 nodes 


````
minikube -p multinode-demo kubectl -- get nodes
````

For conveniency we will install `kubectl` natively

````
sudo snap install kubectl --classic
alias k=`kubectl`
````

It is working as we use deault context setup by minikube at `cat ~/.kube/config`.


### Multinode cluster observations

#### Basic

We can see we now have 2 nodes


````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl get nodes
NAME                 STATUS   ROLES                  AGE   VERSION
multinode-demo       Ready    control-plane,master   18m   v1.23.3
multinode-demo-m02   Ready    <none>                 15m   v1.23.3
````

Create `deloyment.yaml`.

````
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 100%
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      affinity:
        # ⬇⬇⬇ This ensures pods will land on separate hosts
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions: [{ key: app, operator: In, values: [hello] }]
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: hello-from
        image: pbitty/hello-from:latest
        ports:
          - name: http
            containerPort: 80
      terminationGracePeriodSeconds: 1
````

And apply it

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl apply -f deployment.yaml
````

We can see 2 pods have been scheduled on the 2 nodes

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP           NODE                 NOMINATED NODE   READINESS GATES
hello-ccfbb9679-7bnl8   1/1     Running   0          5m59s   10.244.2.2   multinode-demo-m02   <none>           <none>
hello-ccfbb9679-hb5s7   1/1     Running   0          5m59s   10.244.0.3   multinode-demo       <none>           <none>
````


If we check running container 

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED          STATUS          PORTS                                                                                                                                  NAMES
38a2b8271294   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   19 minutes ago   Up 19 minutes   127.0.0.1:49162->22/tcp, 127.0.0.1:49161->2376/tcp, 127.0.0.1:49160->5000/tcp, 127.0.0.1:49159->8443/tcp, 127.0.0.1:49158->32443/tcp   multinode-demo-m02
2bc06cebf32e   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   20 minutes ago   Up 20 minutes   127.0.0.1:49157->22/tcp, 127.0.0.1:49156->2376/tcp, 127.0.0.1:49155->5000/tcp, 127.0.0.1:49154->8443/tcp, 127.0.0.1:49153->32443/tcp   multinode-demo
````

We have 2 running container, which match our 2 nodes.

If we go inside a container (which is here a cluster node), and run `docker ps`, we will `see container running inside docker

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker exec  -it 38a2b8271294 /bin/bash
root@multinode-demo-m02:/# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS     NAMES
1b1d8c66b5df   pbitty/hello-from      "/hello-from"            8 minutes ago    Up 8 minutes              k8s_hello-from_hello-ccfbb9679-7bnl8_default_6958eb79-8430-4fe8-9b64-3a92fefe958b_0
c2dc5b0ada07   k8s.gcr.io/pause:3.6   "/pause"                 9 minutes ago    Up 9 minutes              k8s_POD_hello-ccfbb9679-7bnl8_default_6958eb79-8430-4fe8-9b64-3a92fefe958b_0
22f5a95d26d9   kindest/kindnetd       "/bin/kindnetd"          21 minutes ago   Up 21 minutes             k8s_kindnet-cni_kindnet-9bdrc_kube-system_ce918317-58eb-41d5-af97-ee50765df5b1_0
999ee2ea817b   9b7cc9982109           "/usr/local/bin/kube…"   21 minutes ago   Up 21 minutes             k8s_kube-proxy_kube-proxy-ljc4z_kube-system_5e4f1aaa-cf8a-42f4-b59d-b3e03f5763a1_0
47d1bf9ff4a0   k8s.gcr.io/pause:3.6   "/pause"                 21 minutes ago   Up 21 minutes             k8s_POD_kube-proxy-ljc4z_kube-system_5e4f1aaa-cf8a-42f4-b59d-b3e03f5763a1_0
679daa2d62a0   k8s.gcr.io/pause:3.6   "/pause"                 21 minutes ago   Up 21 minutes             k8s_POD_kindnet-9bdrc_kube-system_ce918317-58eb-41d5-af97-ee50765df5b1_0
root@multinode-demo-m02:/#
````

We have Docker in Docker, where first Docker emulates a cluster node!!

#### We can add a taint to master node to not have POD scheduled on it

````
kubectl taint nodes multinode-demo node-role.kubernetes.io/master:NoSchedule
````

Edit `deloyment.yaml` to remove `podAntiAffinity`.

````
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 100%
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello-from
        image: pbitty/hello-from:latest
        ports:
          - name: http
            containerPort: 80
      terminationGracePeriodSeconds: 1

````
And apply

````
kubectl delete -f deployment.yaml
kubectl apply -f deployment.yaml
````

In that case, we observe pods are not scheduled anymore on the first node.


````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl get pods -o wide
NAME                    READY   STATUS              RESTARTS   AGE   IP       NODE                 NOMINATED NODE   READINESS GATES
hello-99b498c79-6rw2p   0/1     ContainerCreating   0          2s    <none>   multinode-demo-m02   <none>           <none>
hello-99b498c79-9c9rs   0/1     ContainerCreating   0          2s    <none>   multinode-demo-m02   <none>           <none>
hello-99b498c79-gbc9k   0/1     ContainerCreating   0          2s    <none>   multinode-demo-m02   <none>           <none>
hello-99b498c79-q5r5q   0/1     ContainerCreating   0          2s    <none>   multinode-demo-m02   <none>           <none>
hello-99b498c79-qtbr6   0/1     ContainerCreating   0          2s    <none>   multinode-demo-m02   <none>           <none>
---
````

#### Why do we have a pause container?

https://stackoverflow.com/questions/48651269/what-are-the-pause-containers

Delete container and delete pod difference

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker exec  -it 38a2b8271294 /bin/bash
root@multinode-demo-m02:/# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS     NAMES
13ae56dff753   pbitty/hello-from      "/hello-from"            6 hours ago   Up 6 hours             k8s_hello-from_hello-99b498c79-6rw2p_default_d0013b4f-9586-4751-aabf-2c6ef8af34f5_0
1fc32a388ff6   pbitty/hello-from      "/hello-from"            6 hours ago   Up 6 hours             k8s_hello-from_hello-99b498c79-9c9rs_default_c45dfd91-5fc8-4d1b-a1c5-c8aba8bde4f5_0
dd6c84988a36   pbitty/hello-from      "/hello-from"            6 hours ago   Up 6 hours             k8s_hello-from_hello-99b498c79-q5r5q_default_117784a8-8ffd-4821-8345-27fa61069eab_0
9f6b90d03baa   pbitty/hello-from      "/hello-from"            6 hours ago   Up 6 hours             k8s_hello-from_hello-99b498c79-gbc9k_default_e53ed534-21ad-4b34-bf29-18529e259728_0
7bee40d48c3d   pbitty/hello-from      "/hello-from"            6 hours ago   Up 6 hours             k8s_hello-from_hello-99b498c79-qtbr6_default_98243211-6c42-4397-b54b-ac5b7436fae2_0
42efe8c0e641   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago   Up 6 hours             k8s_POD_hello-99b498c79-9c9rs_default_c45dfd91-5fc8-4d1b-a1c5-c8aba8bde4f5_0
5c622d472a45   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago   Up 6 hours             k8s_POD_hello-99b498c79-6rw2p_default_d0013b4f-9586-4751-aabf-2c6ef8af34f5_0
90c3c3ce467d   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago   Up 6 hours             k8s_POD_hello-99b498c79-q5r5q_default_117784a8-8ffd-4821-8345-27fa61069eab_0
24a373788d92   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago   Up 6 hours             k8s_POD_hello-99b498c79-qtbr6_default_98243211-6c42-4397-b54b-ac5b7436fae2_0
e9d2b1c0b094   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago   Up 6 hours             k8s_POD_hello-99b498c79-gbc9k_default_e53ed534-21ad-4b34-bf29-18529e259728_0
22f5a95d26d9   kindest/kindnetd       "/bin/kindnetd"          6 hours ago   Up 6 hours             k8s_kindnet-cni_kindnet-9bdrc_kube-system_ce918317-58eb-41d5-af97-ee50765df5b1_0
999ee2ea817b   9b7cc9982109           "/usr/local/bin/kube…"   6 hours ago   Up 6 hours             k8s_kube-proxy_kube-proxy-ljc4z_kube-system_5e4f1aaa-cf8a-42f4-b59d-b3e03f5763a1_0
47d1bf9ff4a0   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago   Up 6 hours             k8s_POD_kube-proxy-ljc4z_kube-system_5e4f1aaa-cf8a-42f4-b59d-b3e03f5763a1_0
679daa2d62a0   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago   Up 6 hours             k8s_POD_kindnet-9bdrc_kube-system_ce918317-58eb-41d5-af97-ee50765df5b1_0

root@multinode-demo-m02:/# docker kill 1fc32a388ff6
1fc32a388ff6

root@multinode-demo-m02:/# docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS     NAMES
173e49bda96a   pbitty/hello-from      "/hello-from"            5 seconds ago   Up 5 seconds             k8s_hello-from_hello-99b498c79-9c9rs_default_c45dfd91-5fc8-4d1b-a1c5-c8aba8bde4f5_1
13ae56dff753   pbitty/hello-from      "/hello-from"            6 hours ago     Up 6 hours               k8s_hello-from_hello-99b498c79-6rw2p_default_d0013b4f-9586-4751-aabf-2c6ef8af34f5_0
dd6c84988a36   pbitty/hello-from      "/hello-from"            6 hours ago     Up 6 hours               k8s_hello-from_hello-99b498c79-q5r5q_default_117784a8-8ffd-4821-8345-27fa61069eab_0
9f6b90d03baa   pbitty/hello-from      "/hello-from"            6 hours ago     Up 6 hours               k8s_hello-from_hello-99b498c79-gbc9k_default_e53ed534-21ad-4b34-bf29-18529e259728_0
7bee40d48c3d   pbitty/hello-from      "/hello-from"            6 hours ago     Up 6 hours               k8s_hello-from_hello-99b498c79-qtbr6_default_98243211-6c42-4397-b54b-ac5b7436fae2_0
42efe8c0e641   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago     Up 6 hours               k8s_POD_hello-99b498c79-9c9rs_default_c45dfd91-5fc8-4d1b-a1c5-c8aba8bde4f5_0
5c622d472a45   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago     Up 6 hours               k8s_POD_hello-99b498c79-6rw2p_default_d0013b4f-9586-4751-aabf-2c6ef8af34f5_0
90c3c3ce467d   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago     Up 6 hours               k8s_POD_hello-99b498c79-q5r5q_default_117784a8-8ffd-4821-8345-27fa61069eab_0
24a373788d92   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago     Up 6 hours               k8s_POD_hello-99b498c79-qtbr6_default_98243211-6c42-4397-b54b-ac5b7436fae2_0
e9d2b1c0b094   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago     Up 6 hours               k8s_POD_hello-99b498c79-gbc9k_default_e53ed534-21ad-4b34-bf29-18529e259728_0
22f5a95d26d9   kindest/kindnetd       "/bin/kindnetd"          6 hours ago     Up 6 hours               k8s_kindnet-cni_kindnet-9bdrc_kube-system_ce918317-58eb-41d5-af97-ee50765df5b1_0
999ee2ea817b   9b7cc9982109           "/usr/local/bin/kube…"   6 hours ago     Up 6 hours               k8s_kube-proxy_kube-proxy-ljc4z_kube-system_5e4f1aaa-cf8a-42f4-b59d-b3e03f5763a1_0
47d1bf9ff4a0   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago     Up 6 hours               k8s_POD_kube-proxy-ljc4z_kube-system_5e4f1aaa-cf8a-42f4-b59d-b3e03f5763a1_0
679daa2d62a0   k8s.gcr.io/pause:3.6   "/pause"                 6 hours ago     Up 6 hours               k8s_POD_kindnet-9bdrc_kube-system_ce918317-58eb-41d5-af97-ee50765df5b1_0
root@multinode-demo-m02:/# docker kill 173e49bda96a
173e49bda96a
root@multinode-demo-m02:/# exit

````

Container is re-started
See also https://github.com/scoulomb/myk8s/blob/master/Deployment/resilient-deployment.md#actions


````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl get  po -o wide
NAME                    READY   STATUS    RESTARTS        AGE     IP            NODE                 NOMINATED NODE   READINESS GATES
hello-99b498c79-6rw2p   1/1     Running   0               5h36m   10.244.2.10   multinode-demo-m02   <none>           <none>
hello-99b498c79-9c9rs   1/1     Running   2 (2m22s ago)   5h36m   10.244.2.11   multinode-demo-m02   <none>           <none>
hello-99b498c79-gbc9k   1/1     Running   0               5h36m   10.244.2.8    multinode-demo-m02   <none>           <none>
hello-99b498c79-q5r5q   1/1     Running   0               5h36m   10.244.2.9    multinode-demo-m02   <none>           <none>
hello-99b498c79-qtbr6   1/1     Running   0               5h36m   10.244.2.7    multinode-demo-m02   <none>           <none>

scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl delete po hello-99b498c79-9c9rs
pod "hello-99b498c79-9c9rs" deleted

scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl get  po -o wide
NAME                    READY   STATUS              RESTARTS   AGE     IP            NODE                 NOMINATED NODE   READINESS GATES
hello-99b498c79-6rw2p   1/1     Running             0          5h36m   10.244.2.10   multinode-demo-m02   <none>           <none>
hello-99b498c79-gbc9k   1/1     Running             0          5h36m   10.244.2.8    multinode-demo-m02   <none>           <none>
hello-99b498c79-q5r5q   1/1     Running             0          5h36m   10.244.2.9    multinode-demo-m02   <none>           <none>
hello-99b498c79-qtbr6   1/1     Running             0          5h36m   10.244.2.7    multinode-demo-m02   <none>           <none>
hello-99b498c79-z7bph   0/1     ContainerCreating   0          2s      <none>        multinode-demo-m02   <none>           <none>
````

Pod is restarted.



## Alternative to Minikube on machine

### kubeadm

We could here use `kubeadm` (as done here with VM https://github.com/scoulomb/myk8s/tree/master/Setup, https://github.com/scoulomb/myk8s/blob/master/Volumes/non-cloud-volume-additional-appendix.md, and next section [other.md](others.md#use-archlinuxubuntu))...


<!-- setup with Minikube described [](#minikube-docker-in-docker-and-local-multicluster) -->

When using `kubeadm`
1. We install first control plane (master) node node:  https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node (`kubeadm init <args>`).
2. Normally control plane (master) node unlike worker node should not have workload but we can remove taints to allow workload: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation

    > By default, your cluster will not schedule Pods on the control plane nodes for security reasons. If you want to be able to schedule Pods on the control plane nodes, for example for a **single machine Kubernetes cluster**, run:
    
    ````
    kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-`
    ````
    
    > This will remove the node-role.kubernetes.io/control-plane and node-role.kubernetes.io/master taints from any nodes that have them, including the control plane nodes, meaning that the scheduler will then be able to schedule Pods everywhere.

    See [myk8s, kubeadm setup](https://github.com/scoulomb/myk8s/blob/master/Setup/ClusterSetup/removeTaints.sh).

3. We add new node

    From https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes

    Case of worker node

    ````
    kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
    ````

    But new node can also be a master

    https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/


    ````
    sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
    ````

    > The --control-plane flag tells kubeadm join to create a new control plane.


    We can remove taints also on those new master node to allow pods to be scheduled (default behavir different from Minikube on taints).

    This is clear in output of https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#steps-for-the-first-control-plane-node

    ````
    ...
    You can now join any number of control-plane node by running the following command on each as a root:
        kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07

    Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
    As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

    Then you can join any number of worker nodes by running the following on each as root:
        kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
    ````

<!-- --experimental-control-plane in french version -->

### k3s

https://k3s.io/: Lightweigth Kubernetes distribution

In k3s we have server (master) and agent (dedicated worker) node.

k3s allows workload on server node by default.

From https://rancher.com/docs/k3s/latest/en/installation/ha/, https://web.archive.org/web/20220407143515/https://rancher.com/docs/k3s/latest/en/installation/ha/

> By default, server nodes will be schedulable and thus your workloads can get launched on them. If you wish to have a dedicated control plane where no user workloads will run, you can use taints. The node-taint parameter will allow you to configure nodes with taints, for example --node-taint CriticalAddonsOnly=true:NoExecute.

See also https://github.com/k3s-io/k3s/issues/1401

We will first check k3d and setup k3s.
We will not check k3s HA setup (high availability, multinode) on host. But k3s is a particular case of k3s HA.

### k3d


k3d run k3s inside Docker. It is similar to minikube with Docker driver [above](#minikube-docker-in-docker-and-local-multicluster).
See https://blog.gabrielsagnard.fr/gerer-les-clusters-k3s-avec-k3d/



#### Cluster setup


We follow: https://k3d.io/v5.4.4/

````
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | TAG=v5.0.0 bash
````

Note during the setup the context switch but still using: `cat ~/.kube/config`.

````
kubectl config use-context k3d-mycluster
````

Can come back to previous

````
cat ~/.kube/config
kubectl config use-context multinode-demo
````

Then let's create the cluster with 3 nodes ( 1server and 2 agent)

````
k3d cluster create mycluster
````

We can see the cluster

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ k3d cluster list
NAME        SERVERS   AGENTS   LOADBALANCER
mycluster   1/1       0/0      true
````

We will add 2 agent nodes

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ k3d node create -c mycluster jejnode
INFO[0000] Adding 1 node(s) to the runtime local cluster 'mycluster'...
INFO[0000] Starting Node 'k3d-jejnode-0'
INFO[0008] Successfully created 1 node(s)!
````

If we list nodes, we see

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl get nodes
NAME                     STATUS     ROLES                  AGE     VERSION
k3d-mycluster-server-0   Ready      control-plane,master   3m19s   v1.21.5+k3s1
k3d-sysynode-0           Ready      <none>                 18s     v1.21.5+k3s1
k3d-jejnode-0            NotReady   <none>                 4s      v1.21.5+k3s1
````

#### Observations

##### Basic


Apply `deloyment.yaml` with 10 replicas and then 15.

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl apply -f deployment.yaml
````

We can see 15 pods have been scheduled on the 3 nodes

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ k get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE                     NOMINATED NODE   READINESS GATES
hello-7c686947cc-d2qff   1/1     Running   0          35m   10.42.1.3    k3d-sysynode-0           <none>           <none>
hello-7c686947cc-g7n8m   1/1     Running   0          35m   10.42.1.4    k3d-sysynode-0           <none>           <none>
hello-7c686947cc-rmvks   1/1     Running   0          35m   10.42.2.3    k3d-jejnode-0            <none>           <none>
hello-7c686947cc-tkqzm   1/1     Running   0          35m   10.42.2.5    k3d-jejnode-0            <none>           <none>
hello-7c686947cc-vpcdk   1/1     Running   0          35m   10.42.2.4    k3d-jejnode-0            <none>           <none>
hello-7c686947cc-8v5sg   1/1     Running   0          33m   10.42.2.6    k3d-jejnode-0            <none>           <none>
hello-7c686947cc-4klp6   1/1     Running   0          33m   10.42.1.6    k3d-sysynode-0           <none>           <none>
hello-7c686947cc-kmqls   1/1     Running   0          33m   10.42.1.7    k3d-sysynode-0           <none>           <none>
hello-7c686947cc-tctpk   1/1     Running   0          33m   10.42.1.8    k3d-sysynode-0           <none>           <none>
hello-7c686947cc-mzqtz   1/1     Running   0          33m   10.42.1.9    k3d-sysynode-0           <none>           <none>
hello-7c686947cc-bnbq9   1/1     Running   0          33m   10.42.1.5    k3d-sysynode-0           <none>           <none>
hello-7c686947cc-rfzfk   1/1     Running   0          33m   10.42.2.7    k3d-jejnode-0            <none>           <none>
hello-7c686947cc-km66t   1/1     Running   0          33m   10.42.2.8    k3d-jejnode-0            <none>           <none>
hello-7c686947cc-xd7hb   1/1     Running   0          33m   10.42.0.9    k3d-mycluster-server-0   <none>           <none>
hello-7c686947cc-jq7tc   1/1     Running   0          33m   10.42.0.10   k3d-mycluster-server-0   <none>           <none>
````


If we check running container.

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED             STATUS             PORTS                                                                                                                                  NAMES
2b1ce779c510   rancher/k3s:v1.21.5-k3s1              "/bin/k3d-entrypoint…"   37 minutes ago      Up 36 minutes                                                                                                                                             k3d-jejnode-0
2971249926de   rancher/k3s:v1.21.5-k3s1              "/bin/k3d-entrypoint…"   37 minutes ago      Up 37 minutes                                                                                                                                             k3d-sysynode-0
1beafe9a6ddc   rancher/k3d-proxy:5.0.0               "/bin/sh -c nginx-pr…"   40 minutes ago      Up 40 minutes      80/tcp, 0.0.0.0:40683->6443/tcp                                                                                                        k3d-mycluster-serverlb
d706ff21d3d4   rancher/k3s:v1.21.5-k3s1              "/bin/k3d-entrypoint…"   40 minutes ago      Up 40 minutes                                                                                                                                             k3d-mycluster-server-0
38a2b8271294   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   About an hour ago   Up About an hour   127.0.0.1:49162->22/tcp, 127.0.0.1:49161->2376/tcp, 127.0.0.1:49160->5000/tcp, 127.0.0.1:49159->8443/tcp, 127.0.0.1:49158->32443/tcp   multinode-demo-m02
2bc06cebf32e   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   About an hour ago   Up About an hour   127.0.0.1:49157->22/tcp, 127.0.0.1:49156->2376/tcp, 127.0.0.1:49155->5000/tcp, 127.0.0.1:49154->8443/tcp, 127.0.0.1:49153->32443/tcp   multinode-demo
````

 We have 3 docker matching k3d nodes and a looad balancer node.

If we go inside a container (which is here a cluster node), we will see container running inside docker, but Docker is not used (containerd is used)!

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker exec -it 2971249926de /bin/sh
/ # docker ps
/bin/sh: docker: not found

/ # ps
PID   USER     COMMAND
    1 0        /sbin/docker-init -- /bin/k3d-entrypoint.sh agent
    7 0        /bin/k3s agent
   56 0        containerd
  399 0        /bin/containerd-shim-runc-v2 -namespace k8s.io -id 7fb306def3b8022a299f1c43271536ef154e1c9dddfedcf41f89a167447de9f0 -address /run/k3s/containerd/containerd.sock
  418 0        /pause
  464 0        {entry} /bin/sh /usr/bin/entry
  500 0        {entry} /bin/sh /usr/bin/entry
  840 0        /bin/containerd-shim-runc-v2 -namespace k8s.io -id 2f45e26c195472fa69c0de6d0cace0b86e91516a783f2d09678fd41b9a48b12b -address /run/k3s/containerd/containerd.sock
  872 0        /bin/containerd-shim-runc-v2 -namespace k8s.io -id cca4dad645aa4c8a06e2cdbe5edfda43b7b28f657e2fa94a349292a408adcbed -address /run/k3s/containerd/containerd.sock
  888 0        /pause
  899 0        /pause
  981 0        /hello-from
  983 0        /hello-from
 1559 0        /bin/containerd-shim-runc-v2 -namespace k8s.io -id 0993624798625eb7acb7adddfebc2ae12bd0db66efa1d9502d9b645a1b7dc77a -address /run/k3s/containerd/containerd.sock
 1604 0        /pause
 1663 0        /bin/containerd-shim-runc-v2 -namespace k8s.io -id 33c15d2ab82e8f9288179103bc6bc10f27c187309feaa8a011369063b8ed99e2 -address /run/k3s/containerd/containerd.sock
 1709 0        /pause
 1717 0        /bin/containerd-shim-runc-v2 -namespace k8s.io -id 9663e6e87d817e470e7e85ba7e2be6a12a8e38b61a9a78e70603132399b9a914 -address /run/k3s/containerd/containerd.sock
 1778 0        /pause
 1806 0        /bin/containerd-shim-runc-v2 -namespace k8s.io -id 980c7a47863b1dfbf9141c0dca37244e3a92b8aec08beaf4ed68e88b437c7b83 -address /run/k3s/containerd/containerd.sock
 1845 0        /pause
 1859 0        /bin/containerd-shim-runc-v2 -namespace k8s.io -id fea51a0cf49169e0127059b1aaffb144e446dd47be6e13b1568b58fcb9aade36 -address /run/k3s/containerd/containerd.sock
 1908 0        /pause
 1957 0        /hello-from
 1993 0        /hello-from
 2028 0        /hello-from
 2099 0        /hello-from
 2132 0        /hello-from
 9856 0        /bin/sh
 9896 0        ps
/ # ctr container ls
CONTAINER                                                           IMAGE                                                                      RUNTIME
0993624798625eb7acb7adddfebc2ae12bd0db66efa1d9502d9b645a1b7dc77a    docker.io/rancher/pause:3.1                                                io.containerd.runc.v2
1fcc34333fed09a57b58bc46e7e526c9d9d01a95da0ae04dc92715c2f04cdb6d    sha256:cf80e67f50d6580e194521b9d69417b377a949027c9a0f2674368568c644b68c    io.containerd.runc.v2
2629c494afa70a4bd5176c4ae570cff93c56d3afa28753e3be17373f5fdbe369    sha256:cf80e67f50d6580e194521b9d69417b377a949027c9a0f2674368568c644b68c    io.containerd.runc.v2
2b7c3588206096affa34579388a17ea254fc6f61089d24a1b1dbdea3223fd944    docker.io/rancher/klipper-lb:v0.2.0                                        io.containerd.runc.v2
2f45e26c195472fa69c0de6d0cace0b86e91516a783f2d09678fd41b9a48b12b    docker.io/rancher/pause:3.1                                                io.containerd.runc.v2
33c15d2ab82e8f9288179103bc6bc10f27c187309feaa8a011369063b8ed99e2    docker.io/rancher/pause:3.1                                                io.containerd.runc.v2
39dba9b2c9ffc0dba3809f6c4041db8d054f2c6d78a2165fdb9275ea73a42372    sha256:cf80e67f50d6580e194521b9d69417b377a949027c9a0f2674368568c644b68c    io.containerd.runc.v2
4e49863bc665e1f7d2f73b8775937f788be22e2f1e066505acdaf58790a615f2    sha256:cf80e67f50d6580e194521b9d69417b377a949027c9a0f2674368568c644b68c    io.containerd.runc.v2
7338a1e4232e281988b2b4747e5855203a29874fb5a9253684bf8262255be826    sha256:cf80e67f50d6580e194521b9d69417b377a949027c9a0f2674368568c644b68c    io.containerd.runc.v2
7fb306def3b8022a299f1c43271536ef154e1c9dddfedcf41f89a167447de9f0    docker.io/rancher/pause:3.1                                                io.containerd.runc.v2
86a8fdf5aad96a67592045afde53f71baf7fd02c2e490793e8e0e2b979075f28    sha256:cf80e67f50d6580e194521b9d69417b377a949027c9a0f2674368568c644b68c    io.containerd.runc.v2
8899267d139571c7784d4e3964afa51fdffb392b4fd5ee5a5b53bfea159544ce    sha256:cf80e67f50d6580e194521b9d69417b377a949027c9a0f2674368568c644b68c    io.containerd.runc.v2
8c0f7089e87d6587eb2f2481c028e42e28a840451ebc2df62b5057327beb926b    docker.io/rancher/klipper-lb:v0.2.0                                        io.containerd.runc.v2
9663e6e87d817e470e7e85ba7e2be6a12a8e38b61a9a78e70603132399b9a914    docker.io/rancher/pause:3.1                                                io.containerd.runc.v2
980c7a47863b1dfbf9141c0dca37244e3a92b8aec08beaf4ed68e88b437c7b83    docker.io/rancher/pause:3.1                                                io.containerd.runc.v2
cca4dad645aa4c8a06e2cdbe5edfda43b7b28f657e2fa94a349292a408adcbed    docker.io/rancher/pause:3.1                                                io.containerd.runc.v2
fea51a0cf49169e0127059b1aaffb144e446dd47be6e13b1568b58fcb9aade36    docker.io/rancher/pause:3.1                                                io.containerd.runc.v2

/ # ctr container info 4e49863bc665e1f7d2f73b8775937f788be22e2f1e066505acdaf58790a615f2 | grep pod.name
        "io.kubernetes.pod.name": "hello-7c686947cc-bnbq9",
        "io.kubernetes.pod.namespace": "default",

````

We have container in container, where first Docker emulates a cluster node!!


#### Run inside a container in the container

We have 2 ways !

##### Go to node container and from there use container engine

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker exec -it 2971249926de /bin/sh
/ # ctr task exec --tty --exec-id 22221 4e49863bc665e1f7d2f73b8775937f788be22e2f1e066505acdaf58790a615f2 /
bin/sh
/ # ls
bin         hello-from  media       root        srv         usr
dev         home        mnt         run         sys         var
etc         lib         proc        sbin        tmp
/ #
````


Where `2971249926de` is the container id of a node.
And `4e49863bc665e1f7d2f73b8775937f788be22e2f1e066505acdaf58790a615f2` is the container id of container running inside the container node which run hello-from application.

As a reminder we had 

````
/ # ctr container info 4e49863bc665e1f7d2f73b8775937f788be22e2f1e066505acdaf58790a615f2 | grep pod.name
        "io.kubernetes.pod.name": "hello-7c686947cc-bnbq9",
        "io.kubernetes.pod.namespace": "default",
````


##### Use Kubernetes API

The container inside the node container result from the pod.
Therefore we can also use Kubernetes command.


````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ k exec -it hello-7c686947cc-bnbq9 -- /bin/sh
/ # ls
bin         hello-from  media       root        srv         usr
dev         home        mnt         run         sys         var
etc         lib         proc        sbin        tmp
/ #
````


#### Come back to hp machine

We can see all the process are actually running on the host machine

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo ps -aux | grep hello
root       92500  0.0  0.0   7044  3928 ?        Ssl  17:38   0:00 /hello-from
root       92563  0.0  0.0   7044  3988 ?        Ssl  17:38   0:00 /hello-from
root       92625  0.0  0.0   9508  3984 ?        Ssl  17:38   0:00 /hello-from
root       92697  0.0  0.0   9508  4156 ?        Ssl  17:38   0:00 /hello-from
root       92760  0.0  0.0   8100  3980 ?        Ssl  17:38   0:00 /hello-from
root      106563  0.0  0.0   7044  3972 ?        Ssl  17:50   0:00 /hello-from
root      106565  0.0  0.0   7044  3928 ?        Ssl  17:50   0:00 /hello-from
root      106950  0.0  0.0   8100  3984 ?        Ssl  17:50   0:00 /hello-from
root      106966  0.0  0.0   7044  3932 ?        Ssl  17:50   0:00 /hello-from
root      107007  0.0  0.0   8100  3924 ?        Ssl  17:50   0:00 /hello-from
root      111013  0.0  0.0   7044  3928 ?        Ssl  17:53   0:00 /hello-from
root      111021  0.0  0.0   7044  3988 ?        Ssl  17:53   0:00 /hello-from
root      111070  0.0  0.0   8100  3980 ?        Ssl  17:53   0:00 /hello-from
root      111120  0.0  0.0   7044  3984 ?        Ssl  17:53   0:00 /hello-from
root      111122  0.0  0.0   7044  3932 ?        Ssl  17:53   0:00 /hello-from
root      111301  0.0  0.0   7044  3928 ?        Ssl  17:53   0:00 /hello-from
root      111332  0.0  0.0   7044  3988 ?        Ssl  17:53   0:00 /hello-from
root        0.0  0.0   7044  3924 ?        Ssl  17:53   0:00 /hello-from
root      112066  0.0  0.0   7044  3788 ?        Ssl  17:53   0:00 /hello-from
root      112067  0.0  0.0   6952  3848 ?        Ssl  17:53   0:00 /hello-from
scoulomb  175338  0.0  0.0   9264  2316 pts/4    S+   18:44   0:00 grep --color=auto hello

scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo ps -aux | grep hello | wc -l
21
````

We have 15 container from k3d test (17:50, 17:53) and 5 from minikube (17:38).

Let's take a container from k3d and find who triggered it

We will go through ppid chain. 

We choose `111369`

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /proc/111369/status | grep PPid
PPid:   110821
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /proc/110821/status | grep PPid
PPid:   102794
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /proc/102794/status | grep PPid
PPid:   102773
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /proc/102773/status | grep PPid
PPid:   1


scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /proc/111369/cmdline | tr '\0' " "
/hello-from 
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /proc/110821/cmdline | tr '\0' " "
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /proc/102794/cmdline | tr '\0' " "
/sbin/docker-init -- /bin/k3d-entrypoint.sh agent 
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /proc/102773/cmdline | tr '\0' " "
/usr/bin/containerd-shim-runc-v2 -namespace moby -id 2971249926dee369f95bc89790937175c7099c0d93be80b62eeafc2930f4383a -address /run/containerd/containerd.sock scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /proc/1/cmdline | tr '\0' " "
/sbin/init splash 
````

Or do 


````
$ pstree > outpstree.txt
$ ps -aef --forest | grep -C 10 111369

````

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ ps -aef --forest | grep -B 1 -A 5 "111369"
scoulomb  492454  430659  0 23:08 pts/2    00:00:00              \_ ps -aef --forest
scoulomb  492455  430659  0 23:08 pts/2    00:00:00              \_ grep --color=auto -B 1 -A 5 111369
root        1032       1  0 juil.14 ?      00:00:00 /usr/sbin/gdm3
root        1041    1032  0 juil.14 ?      00:00:00  \_ gdm-session-worker [pam/gdm-autologin]
scoulomb    1172    1041  0 juil.14 tty2   00:00:00      \_ /usr/libexec/gdm-wayland-session env GNOME_SHELL_SESSION_MODE=ubuntu /usr/bin/gnome-session --session=ubuntu
scoulomb    1186    1172  0 juil.14 tty2   00:00:00          \_ /usr/libexec/gnome-session-binary --systemd --session=ubuntu
scoulomb    1066       1  0 juil.14 ?      00:00:11 /lib/systemd/systemd --user
--
root      110905  110821  0 17:53 ?        00:00:00          \_ /pause
root      111369  110821  0 17:53 ?        00:00:00          \_ /hello-from
root      103393       1  0 17:49 ?        00:00:03 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 2b1ce779c510c9171073b90ed1f045c049619672853c26245ebd9b1c22a12ba4 -address /run/containerd/containerd.sock
root      103416  103393  0 17:49 ?        00:00:01  \_ /sbin/docker-init -- /bin/k3d-entrypoint.sh agent
root      103476  103416  5 17:49 ?        00:18:03      \_ /bin/k3s agent
root      103526  103476  1 17:49 ?        00:04:13      |   \_ containerd
root      104750  103416  0 17:49 ?        00:00:20      \_ /bin/containerd-shim-runc-v2 -namespace k8s.io -id 898f950dd3a5a27fc4a57aea6be2b3243e8d0b13926f7df4049fb8f38108fc6f -address /run/k3s/containerd/containerd.sock
````



#### We can add a taint to master node to not have POD scheduled on it

````
kubectl taint nodes k3d-mycluster-server-0 node-role.kubernetes.io/master:NoSchedule
cp deployment.yaml deployment2.yaml
# edit deployment hello -> hello2 and 50 replicas
kubectl apply -f deployment2.yaml
kubectl get pods -o wide | grep hello-2 # nothing scheduled on server
kubectl delete -f deployment2.yaml
kubectl taint nodes k3d-mycluster-server-0 node-role.kubernetes.io/master:NoSchedule- # node/k3d-mycluster-server-0 untainted
kubectl apply -f deployment2.yaml # 2 # pod secduled scheduled on server
````


### Setup k3s on machine



Setup k3s in single node

From: https://rancher.com/docs/k3s/latest/en/quick-start/

````
curl -sfL https://get.k3s.io | sh -
# Check for Ready node,
takes maybe 30 seconds
sudo k3s kubectl get node
sudo systemctl status k3s
````

If we do not have nodes and `sudo systemctl status k3s` returns

````
juil. 15 23:41:39 scoulomb-HP-Pavilion-TS-Sleekbook-14 k3s[569842]: time="2022-07-15T23:41:39+02:00" level=info msg="Waiting for control-plane node scoulomb-hp-pavilion-ts-sleekbook-14 startup: nodes \"scoulomb>
juil. 15 23:41:39 scoulomb-HP-Pavilion-TS-Sleekbook-14 k3s[569842]: time="2022-07-15T23:41:39+02:00" level=info msg="Waiting for containerd startup: rpc error: code = Unimplemented desc = unknown service runtim>
jailed to watch cert and key file, will retry later" err="error creating fsnoti>
````

We actually need to remove k3d cluster

````
k3d cluster stop mycluster
sudo systemctl restart k3s
````

Here we can see our node is the host machine (which can be a (VirtualBox) VM)

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo k3s kubectl get node
NAME                                   STATUS   ROLES                  AGE     VERSION
scoulomb-hp-pavilion-ts-sleekbook-14   Ready    control-plane,master   7m48s   v1.23.8+k3s2
````

And can deploy

````
sudo k3s kubectl apply -f deployment.yaml
````
`
And can see pods

````
hello-99b498c79-97jgx   0/1     ContainerCreating   0          29s   <none>   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo k3s kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE                                   NOMINATED NODE   READINESS GATES
hello-99b498c79-xk9kg   1/1     Running   0          41s   10.42.0.14   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-rpm79   1/1     Running   0          41s   10.42.0.15   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-xmr4m   1/1     Running   0          40s   10.42.0.13   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-lhs4m   1/1     Running   0          40s   10.42.0.16   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-97jgx   1/1     Running   0          40s   10.42.0.23   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-ptg5f   1/1     Running   0          41s   10.42.0.10   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-8k4mt   1/1     Running   0          40s   10.42.0.22   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-ks5vm   1/1     Running   0          40s   10.42.0.19   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-6dspr   1/1     Running   0          41s   10.42.0.17   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-jmd9p   1/1     Running   0          41s   10.42.0.12   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-sxbq2   1/1     Running   0          41s   10.42.0.11   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-bnrn9   1/1     Running   0          40s   10.42.0.18   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-kmnqt   1/1     Running   0          41s   10.42.0.9    scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-l49wb   1/1     Running   0          40s   10.42.0.21   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
hello-99b498c79-lfcpt   1/1     Running   0          40s   10.42.0.20   scoulomb-hp-pavilion-ts-sleekbook-14   <none>           <none>
````

And can access images

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED       STATUS       PORTS                                                                                                                                  NAMES
38a2b8271294   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   7 hours ago   Up 7 hours   127.0.0.1:49162->22/tcp, 127.0.0.1:49161->2376/tcp, 127.0.0.1:49160->5000/tcp, 127.0.0.1:49159->8443/tcp, 127.0.0.1:49158->32443/tcp   multinode-demo-m02
2bc06cebf32e   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   7 hours ago   Up 7 hours   127.0.0.1:49157->22/tcp, 127.0.0.1:49156->2376/tcp, 127.0.0.1:49155->5000/tcp, 127.0.0.1:49154->8443/tcp, 127.0.0.1:49153->32443/tcp   multinode-demo
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo ctr list
No help topic for 'list'
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo ctr container list
CONTAINER    IMAGE    RUNTIME
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo k3s ctr container list
CONTAINER                                                           IMAGE                                                  RUNTIME
017742f4854ee124213ac433319dfa04350986570a5ee76baf052ead1bb39627    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
08c8a23a99e7c1cbcc13caae70fd37628c092d71088268c0df74e8e5235679a7    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
0c07838049d1ab3dd898b6433cb342d4f7487ec0b6b9b764b72bd5716892ea47    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
1808b380ef96063e731de429114f34549697d6d2e766c0703bacfb8e895233e4    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
1b993552712b74da0055a957485153fff4471f24b06684c8b668f28f93efac85    docker.io/rancher/mirrored-coredns-coredns:1.9.1       io.containerd.runc.v2
2045333475646b734225354ee96f614236155e1b74534d18075692e523409ce7    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
22efbff95a022866dff23b53aea679c74c5b06abd0cd579c39110b594762c592    docker.io/rancher/klipper-lb:v0.3.5                    io.containerd.runc.v2
231a848e53e54fd29d20578c503b0313f960c13fac5a07ea81b3e38c1f531e60    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
2b8e64fce7fe51893d0a16fc029ea81dd249b3afb618e4ae91f87afaa28132d3    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
368744a50e9d11864b13ba26f419b9dec674f206939d84a156fecc4980e6a142    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
3c29bc630f33267cca39cc6e6de2c23cf448804e6ae0a0bebd074d96b66e9f3e    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
3e98b9544278c35c98bd79ebe121693be394b56c2e4f0644756e5c2c6c36ec38    docker.io/rancher/klipper-lb:v0.3.5                    io.containerd.runc.v2
4190ee036dad4e30200f43ee7bbe4bec75d182567224678aa80130e5c4824fb2    docker.io/rancher/klipper-helm:v0.7.3-build20220613    io.containerd.runc.v2
4b748b35999d75f7c5cf3114eb7a370d42555780ddd16eb2dd6e28fe2a5f3638    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
4e2b1daba78236b2e4df73136c2477049c11187b1be1a07b7b9e091e48db7703    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
4f3b566226be2cd9de7eef6af32f28451bd4509415314e66f9413bb32f6c1de8    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
5832f94ad3ee79cc0344d7bffe534f9bf0ed93a8c49649862d5292cdc2a4afff    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
5bc604cfd3eb14568402de7b4938d7912046266beca3b7d3e443c10f05b43580    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
63b038a9ded9eda5a7a67a78ace1aeaf0cd05c5cc6ed0762a2285b2d802f9a00    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
6410363571b5753315836156cb57cbb01f17060a18ca32401ddeb1c13dde7797    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
66bb756c82db3037c49df02edba344c104f7066eb77795b8b00d1395036ea8a4    docker.io/rancher/local-path-provisioner:v0.0.21       io.containerd.runc.v2
67cf4fa1e61c4076f8c4a1dc82fee55ea79039b71e17dd4124e9677219c727d3    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
6964aef8f27c0496858d8a90aabcd37102c25f919d1156136373ff58cb13b100    docker.io/rancher/mirrored-library-traefik:2.6.2       io.containerd.runc.v2
725bf3f9f996451283c1b9c4760ec2f6a8677678edb80160949a79655a95cc3e    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
788432242bf2dc9151cbc09123fe03504bd19adf44e8dd8e08f3ed0028617ff1    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
78d5ab6871e425abaeabab539c2535f3aaa006a8b24ff945417320f43888f7c3    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
7ced9a8b3b8d11584b5e54cab4e89ee19699968ae8494f88293e234774654e8c    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
822977d2d892bc4d71ccea7207f460fb2362a5ffd0dd3bf3e95e3ec74dd52610    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
888698303bd20eb3d236a3b7b055787608a3d82e36aa69b15025d260a913d38c    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
8d400b2387a6d67cb83a56f3fc7af2e8f5104afde0e76e4f3bc10e9c7c9823d0    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
9783421a83fa47ba63f72db488ff8982e9bda982b29b6cdb57ddf8fe671571b4    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
9854d4bc5f2f8170bdbcb7f09ad656ccaa682a5026e5d11a1954e5b59237755f    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
a34949890302909b03d442da498444697d48e99516343d3b77c8be3311fb3883    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
a77dbf6f1e88f527b8b5c3c43bc18ae2d0575a910c83e2e9399855bc25bcd94e    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
b2fa4717d493f9ef227057152c8cc5b20b84f477d04ba9e2b17c7075f317d718    docker.io/rancher/mirrored-metrics-server:v0.5.2       io.containerd.runc.v2
bc2e3d85544ba3b1a204359af478905542871df489619e13b188591ccd8d451a    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
c3b72bfddd5f4b92de900647b5bf01a9386487eab403ed6f585f3d1cc2148c8a    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
ca57d8fe50e8b43464edb90613fea8924126fe7e6d279b35350a57dd227dcc31    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
cb4d846bf1e5955bb9b43bfe9894edfadc4f547134e1e9b3cd9c95c13357ba2d    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
d046164cfb4c8ae0173c3124fead915a5c9465f66be2df9e7a896fe81b1746de    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
dc148dac0425e3f1b32209673e5dc97a2239ba8948c124d4b68d7a727e7266c3    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
ec730b7b82ff5924e6b642b84b1596576bb470b535aa65ec5df9819b7dceee8e    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
f4abed9c5f3243d807be104eaf53aa44885eec28b35505e0b3ebf1340f9fe1e5    docker.io/rancher/klipper-helm:v0.7.3-build20220613    io.containerd.runc.v2
fbb8bcaadb43d18483799495844adde7dee5486a715d565d7dc66ce05d2a4b31    docker.io/pbitty/hello-from:latest                     io.containerd.runc.v2
fbecd99b8782937daa9cca3272bca205faae2cb776980bb1c0143c3afb810909    docker.io/rancher/mirrored-pause:3.6                   io.containerd.runc.v2
````

But no via Docker!!
Only via `k3s containerd`

Then as in  k3d we can access inside container in 2 ways 

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo k3s ctr container info 08c8a23a99e7c1cbcc13caae70fd37628c092d71088268c0df74e8e5235679a7 | grep pod.name
        "io.kubernetes.pod.name": "hello-99b498c79-bnrn9",
        "io.kubernetes.pod.namespace": "default",
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo k3s ctr task exec --tty --exec-id 22221 08c8a23a99e7c1cbcc13caae70fd37628c092d71088268c0df74e8e5235679a7  /bin/sh
/ # ls
bin         hello-from  media       root        srv         usr
dev         home        mnt         run         sys         var
etc         lib         proc        sbin        tmp


/ # scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo k3s kubectl get po
NAME                    READY   STATUS    RESTARTS   AGE
hello-99b498c79-xk9kg   1/1     Running   0          16m
hello-99b498c79-rpm79   1/1     Running   0          16m
hello-99b498c79-xmr4m   1/1     Running   0          16m
hello-99b498c79-lhs4m   1/1     Running   0          16m
hello-99b498c79-97jgx   1/1     Running   0          16m
hello-99b498c79-ptg5f   1/1     Running   0          16m
hello-99b498c79-8k4mt   1/1     Running   0          16m
hello-99b498c79-ks5vm   1/1     Running   0          16m
hello-99b498c79-6dspr   1/1     Running   0          16m
hello-99b498c79-jmd9p   1/1     Running   0          16m
hello-99b498c79-sxbq2   1/1     Running   0          16m
hello-99b498c79-bnrn9   1/1     Running   0          16m
hello-99b498c79-kmnqt   1/1     Running   0          16m
hello-99b498c79-l49wb   1/1     Running   0          16m
hello-99b498c79-lfcpt   1/1     Running   0          16m
/ # scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo k3s kubectl exec -it "hello-99b498c79-bnrn9" -- /bin/sh
/ # ls
bin         dev         etc         hello-from  home        lib         media       mnt         proc        root        run         sbin        srv         sys         tmp         usr         var
/ #
````

To configure kubectl direct access. This time it is not using `cat ~/.kube/config`

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat ~/.kube/config  | grep k3s
````

See https://rancher.com/docs/k3s/latest/en/cluster-access/


Use `/etc/rancher/k3s/k3s.yaml`

Do 

````
sudo kubectl --kubeconfig /etc/rancher/k3s/k3s.yaml get pods --all-namespaces
````
or with `KUBECONFIG` [env var](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/#the-kubeconfig-environment-variable)

````
sudo su
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes
kubectl get pods --all-namespaces
kubectl config view
````

Output is 

````
root@scoulomb-HP-Pavilion-TS-Sleekbook-14:/home/scoulomb# kubectl get nodes
NAME                                   STATUS   ROLES                  AGE   VERSION
scoulomb-hp-pavilion-ts-sleekbook-14   Ready    control-plane,master   14h   v1.23.8+k3s2
scoulomb-hp-pavilion-ts-sleekbook-14   Ready    control-plane,master   14h   v1.23.8+k3s2
root@scoulomb-HP-Pavilion-TS-Sleekbook-14:/home/scoulomb# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
````

Note the current context, which we can switch: https://github.com/scoulomb/myk8s/blob/master/Master-Kubectl/kube-config.md 


## About containerd and Docker

See https://sweetcode.io/getting-started-with-containerd/

See notes here: https://github.com/scoulomb/myk8s/blob/master/container-engine/container-engine.md

<!-- here all ccl stop, do not go further, YES CCL :)-->


## Comment on master nodes

<details>
  <summary>Click to expand!</summary>
  

master/server nodes describe taints but also deploys more kube-system container, or process for scheduler, etcd...

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED        STATUS        PORTS                                                                                                                                  NAMES
38a2b8271294   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   20 hours ago   Up 20 hours   127.0.0.1:49162->22/tcp, 127.0.0.1:49161->2376/tcp, 127.0.0.1:49160->5000/tcp, 127.0.0.1:49159->8443/tcp, 127.0.0.1:49158->32443/tcp   multinode-demo-m02
2bc06cebf32e   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   20 hours ago   Up 20 hours   127.0.0.1:49157->22/tcp, 127.0.0.1:49156->2376/tcp, 127.0.0.1:49155->5000/tcp, 127.0.0.1:49154->8443/tcp, 127.0.0.1:49153->32443/tcp   multinode-demo
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker exec -it 38a2b8271294  docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED        STATUS        PORTS     NAMES
be9ce51bf22e   pbitty/hello-from      "/hello-from"            14 hours ago   Up 14 hours             k8s_hello-from_hello-99b498c79-z7bph_default_e25b13a7-ea14-4c22-ac61-eee431b1e36e_0
cc80f68e6ffe   k8s.gcr.io/pause:3.6   "/pause"                 14 hours ago   Up 14 hours             k8s_POD_hello-99b498c79-z7bph_default_e25b13a7-ea14-4c22-ac61-eee431b1e36e_0
13ae56dff753   pbitty/hello-from      "/hello-from"            20 hours ago   Up 20 hours             k8s_hello-from_hello-99b498c79-6rw2p_default_d0013b4f-9586-4751-aabf-2c6ef8af34f5_0
dd6c84988a36   pbitty/hello-from      "/hello-from"            20 hours ago   Up 20 hours             k8s_hello-from_hello-99b498c79-q5r5q_default_117784a8-8ffd-4821-8345-27fa61069eab_0
9f6b90d03baa   pbitty/hello-from      "/hello-from"            20 hours ago   Up 20 hours             k8s_hello-from_hello-99b498c79-gbc9k_default_e53ed534-21ad-4b34-bf29-18529e259728_0
7bee40d48c3d   pbitty/hello-from      "/hello-from"            20 hours ago   Up 20 hours             k8s_hello-from_hello-99b498c79-qtbr6_default_98243211-6c42-4397-b54b-ac5b7436fae2_0
5c622d472a45   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_hello-99b498c79-6rw2p_default_d0013b4f-9586-4751-aabf-2c6ef8af34f5_0
90c3c3ce467d   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_hello-99b498c79-q5r5q_default_117784a8-8ffd-4821-8345-27fa61069eab_0
24a373788d92   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_hello-99b498c79-qtbr6_default_98243211-6c42-4397-b54b-ac5b7436fae2_0
e9d2b1c0b094   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_hello-99b498c79-gbc9k_default_e53ed534-21ad-4b34-bf29-18529e259728_0
22f5a95d26d9   kindest/kindnetd       "/bin/kindnetd"          20 hours ago   Up 20 hours             k8s_kindnet-cni_kindnet-9bdrc_kube-system_ce918317-58eb-41d5-af97-ee50765df5b1_0
999ee2ea817b   9b7cc9982109           "/usr/local/bin/kube…"   20 hours ago   Up 20 hours             k8s_kube-proxy_kube-proxy-ljc4z_kube-system_5e4f1aaa-cf8a-42f4-b59d-b3e03f5763a1_0
47d1bf9ff4a0   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_kube-proxy-ljc4z_kube-system_5e4f1aaa-cf8a-42f4-b59d-b3e03f5763a1_0
679daa2d62a0   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_kindnet-9bdrc_kube-system_ce918317-58eb-41d5-af97-ee50765df5b1_0
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker exec -it 2bc06cebf32e  docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED        STATUS        PORTS     NAMES
809b4bfacfe5   a4ca41631cc7           "/coredns -conf /etc…"   20 hours ago   Up 20 hours             k8s_coredns_coredns-64897985d-5nwwf_kube-system_045a0277-a063-4f4f-af4f-96f65d5d04e5_0
3416f943b065   6e38f40d628d           "/storage-provisioner"   20 hours ago   Up 20 hours             k8s_storage-provisioner_storage-provisioner_kube-system_b926c722-1029-419e-84bf-acacf70fcdd0_0
6225faacc4bb   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_storage-provisioner_kube-system_b926c722-1029-419e-84bf-acacf70fcdd0_0
2f11e4ab124b   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_coredns-64897985d-5nwwf_kube-system_045a0277-a063-4f4f-af4f-96f65d5d04e5_0
ce4eb1560d9a   kindest/kindnetd       "/bin/kindnetd"          20 hours ago   Up 20 hours             k8s_kindnet-cni_kindnet-xnp7f_kube-system_24b4a1ff-28a0-436d-8e09-22c8bfd8ff05_0
9cc172968292   9b7cc9982109           "/usr/local/bin/kube…"   20 hours ago   Up 20 hours             k8s_kube-proxy_kube-proxy-pbh98_kube-system_06605907-82c0-482a-9a41-32136630d39d_0
fcb13b4a27d8   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_kindnet-xnp7f_kube-system_24b4a1ff-28a0-436d-8e09-22c8bfd8ff05_0
d73061dcf175   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_kube-proxy-pbh98_kube-system_06605907-82c0-482a-9a41-32136630d39d_0
ceb178c48372   b07520cd7ab7           "kube-controller-man…"   20 hours ago   Up 20 hours             k8s_kube-controller-manager_kube-controller-manager-multinode-demo_kube-system_f6e400c2dff7528fac09fe362ce2ac86_0
d00435c11653   25f8c7f3da61           "etcd --advertise-cl…"   20 hours ago   Up 20 hours             k8s_etcd_etcd-multinode-demo_kube-system_af3f525264cc2334f6ab5e70816db6a8_0
f4055ecfa874   f40be0088a83           "kube-apiserver --ad…"   20 hours ago   Up 20 hours             k8s_kube-apiserver_kube-apiserver-multinode-demo_kube-system_a9c716a303f1c8ad2d57951baace18ad_0
8acb2a7ec45e   99a3486be4f2           "kube-scheduler --au…"   20 hours ago   Up 20 hours             k8s_kube-scheduler_kube-scheduler-multinode-demo_kube-system_ceb90fa1d01210a988ff14a530d965dc_0
e7b7cc4f872b   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_kube-apiserver-multinode-demo_kube-system_a9c716a303f1c8ad2d57951baace18ad_0
cbe9968ef5ed   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_etcd-multinode-demo_kube-system_af3f525264cc2334f6ab5e70816db6a8_0
12d0c01d4ffa   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_kube-controller-manager-multinode-demo_kube-system_f6e400c2dff7528fac09fe362ce2ac86_0
1eb79058b589   k8s.gcr.io/pause:3.6   "/pause"                 20 hours ago   Up 20 hours             k8s_POD_kube-scheduler-multinode-demo_kube-system_ceb90fa1d01210a988ff14a530d965dc_0

````

See https://livebook.manning.com/book/kubernetes-in-action/chapter-17/151 (figure 1.9)
Actually master node has all elememt of worker node (it needs kubelet + container runtime to run kube-system and needs kube-proxy for  network).

We can see master has etcd, apiserver, scheduler etc

See also process view

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ minikube start --nodes 2 -p multinode-demo --driver=docker
😄  [multinode-demo] minikube v1.25.2 on Ubuntu 21.10

[....] restart but did not change node, reconfigure kubectl 

scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl get po -o wide -n kube-system
NAME                                     READY   STATUS    RESTARTS   AGE   IP             NODE                 NOMINATED NODE   READINESS GATES
coredns-64897985d-5nwwf                  1/1     Running   0          20h   10.244.0.2     multinode-demo       <none>           <none>
etcd-multinode-demo                      1/1     Running   0          20h   192.168.58.2   multinode-demo       <none>           <none>
kindnet-9bdrc                            1/1     Running   0          20h   192.168.58.3   multinode-demo-m02   <none>           <none>
kindnet-xnp7f                            1/1     Running   0          20h   192.168.58.2   multinode-demo       <none>           <none>
kube-apiserver-multinode-demo            1/1     Running   0          20h   192.168.58.2   multinode-demo       <none>           <none>
kube-controller-manager-multinode-demo   1/1     Running   0          20h   192.168.58.2   multinode-demo       <none>           <none>
kube-proxy-ljc4z                         1/1     Running   0          20h   192.168.58.3   multinode-demo-m02   <none>           <none>
kube-proxy-pbh98                         1/1     Running   0          20h   192.168.58.2   multinode-demo       <none>           <none>
kube-scheduler-multinode-demo            1/1     Running   0          20h   192.168.58.2   multinode-demo       <none>           <none>
storage-provisioner                      1/1     Running   0          20h   192.168.58.2   multinode-demo       <none>           <none>

scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED        STATUS        PORTS                                                                                                                                  NAMES
38a2b8271294   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   21 hours ago   Up 21 hours   127.0.0.1:49162->22/tcp, 127.0.0.1:49161->2376/tcp, 127.0.0.1:49160->5000/tcp, 127.0.0.1:49159->8443/tcp, 127.0.0.1:49158->32443/tcp   multinode-demo-m02
2bc06cebf32e   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   21 hours ago   Up 21 hours   127.0.0.1:49157->22/tcp, 127.0.0.1:49156->2376/tcp, 127.0.0.1:49155->5000/tcp, 127.0.0.1:49154->8443/tcp, 127.0.0.1:49153->32443/tcp   multinode-demo
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker exec -it 38a2b8271294  ps -aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.1  22508 11480 ?        Ss   Jul15   0:05 /sbin/init
root         103  0.0  0.1  28908  9756 ?        S<s  Jul15   0:00 /lib/systemd/
message+     122  0.0  0.0   7172  2836 ?        Ss   Jul15   0:01 /usr/bin/dbus
root         125  0.1  0.5 1419808 47376 ?       Ssl  Jul15   1:16 /usr/bin/cont
root         139  0.0  0.0  12184  6604 ?        Ss   Jul15   0:00 sshd: /usr/sb
root         379  1.3  1.0 1753248 83296 ?       Ssl  Jul15  16:56 /usr/bin/dock
root        1040  0.0  0.1 711448  9340 ?        Sl   Jul15   0:05 /usr/bin/cont
root        1066  0.0  0.1 711192  8924 ?        Sl   Jul15   0:04 /usr/bin/cont
65535       1090  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
65535       1097  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
root        1131  0.0  0.1 711448  8212 ?        Sl   Jul15   0:04 /usr/bin/cont
root        1151  0.0  0.4 748440 39504 ?        Ssl  Jul15   0:15 /usr/local/bi
root        1516  0.0  0.1 712856  9240 ?        Sl   Jul15   0:06 /usr/bin/cont
root        1536  0.0  0.3 732608 27244 ?        Ssl  Jul15   0:31 /bin/kindnetd
root      179437  5.5  1.2 2084260 99752 ?       Ssl  11:30   0:14 /var/lib/mini
root      179759  0.0  0.0 711448  7544 ?        Sl   11:30   0:00 /usr/bin/cont
root      179782  0.0  0.0 712856  7776 ?        Sl   11:30   0:00 /usr/bin/cont
65535     179806  0.0  0.0    972     4 ?        Ss   11:30   0:00 /pause
65535     179809  0.0  0.0    972     4 ?        Ss   11:30   0:00 /pause
root      179881  0.0  0.1 712856  8068 ?        Sl   11:30   0:00 /usr/bin/cont
65535     179936  0.0  0.0    972     4 ?        Ss   11:30   0:00 /pause
root      179989  0.0  0.0 713112  7812 ?        Sl   11:30   0:00 /usr/bin/cont
root      180014  0.0  0.1 711192  8740 ?        Sl   11:30   0:00 /usr/bin/cont
65535     180055  0.0  0.0    972     4 ?        Ss   11:30   0:00 /pause
65535     180064  0.0  0.0    972     4 ?        Ss   11:30   0:00 /pause
root      180210  0.0  0.1 711448  8812 ?        Sl   11:30   0:00 /usr/bin/cont
root      180231  0.0  0.0   7044  3932 ?        Ssl  11:30   0:00 /hello-from
root      180276  0.0  0.1 712856  8224 ?        Sl   11:30   0:00 /usr/bin/cont
root      180296  0.0  0.0   7044  3984 ?        Ssl  11:30   0:00 /hello-from
root      180323  0.0  0.1 712600  8080 ?        Sl   11:30   0:00 /usr/bin/cont
root      180345  0.0  0.0   7044  3988 ?        Ssl  11:30   0:00 /hello-from
root      180373  0.0  0.1 711448  9280 ?        Sl   11:30   0:00 /usr/bin/cont
root      180394  0.0  0.0   8100  3988 ?        Ssl  11:30   0:00 /hello-from
root      180448  0.0  0.1 711704  8336 ?        Sl   11:30   0:00 /usr/bin/cont
root      180468  0.0  0.0   7044  3984 ?        Ssl  11:30   0:00 /hello-from
root      181846  0.0  0.0   5904  2900 pts/1    Rs+  11:34   0:00 ps -aux
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker exec -it 2bc06cebf32e  ps -aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.1  22436 11432 ?        Ss   Jul15   0:04 /sbin/init
root         101  0.0  0.1  29080 10364 ?        S<s  Jul15   0:00 /lib/systemd/
message+     120  0.0  0.0   7152  2740 ?        Ss   Jul15   0:01 /usr/bin/dbus
root         124  0.0  0.5 1567016 46624 ?       Ssl  Jul15   0:43 /usr/bin/cont
root         133  0.0  0.0  12184  6596 ?        Ss   Jul15   0:00 sshd: /usr/sb
root         378  1.2  1.0 1901224 82108 ?       Ssl  Jul15  14:54 /usr/bin/dock
root        1219  0.0  0.1 711448  8984 ?        Sl   Jul15   0:04 /usr/bin/cont
root        1220  0.0  0.1 712664  9024 ?        Sl   Jul15   0:03 /usr/bin/cont
root        1221  0.0  0.1 711448  8820 ?        Sl   Jul15   0:03 /usr/bin/cont
root        1222  0.0  0.1 711448  9036 ?        Sl   Jul15   0:04 /usr/bin/cont
65535       1296  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
65535       1304  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
65535       1310  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
65535       1312  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
root        1381  0.0  0.0 713112  7724 ?        Sl   Jul15   0:04 /usr/bin/cont
root        1402  0.2  0.6 754804 52860 ?        Ssl  Jul15   3:20 kube-schedule
root        1427  0.0  0.1 713112  8120 ?        Sl   Jul15   0:05 /usr/bin/cont
root        1452  5.7  4.0 1111880 324236 ?      Ssl  Jul15  71:19 kube-apiserve
root        1471  0.0  0.1 711704  8372 ?        Sl   Jul15   0:04 /usr/bin/cont
root        1477  0.0  0.0 712856  8012 ?        Sl   Jul15   0:04 /usr/bin/cont
root        1509  1.7  0.9 11215300 76148 ?      Ssl  Jul15  22:00 etcd --advert
root        1516  2.0  1.3 825876 107888 ?       Ssl  Jul15  25:08 kube-controll
root        1730  3.2  1.3 2085540 105648 ?      Ssl  Jul15  39:57 /var/lib/mini
root        2064  0.0  0.1 711448  8080 ?        Sl   Jul15   0:04 /usr/bin/cont
root        2080  0.0  0.1 712856  8408 ?        Sl   Jul15   0:04 /usr/bin/cont
65535       2110  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
65535       2118  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
root        2155  0.0  0.1 712856  9196 ?        Sl   Jul15   0:04 /usr/bin/cont
root        2175  0.0  0.4 748440 39648 ?        Ssl  Jul15   0:15 /usr/local/bi
root        2550  0.0  0.1 711448  8944 ?        Sl   Jul15   0:06 /usr/bin/cont
root        2571  0.0  0.3 732608 27392 ?        Ssl  Jul15   0:28 /bin/kindnetd
root        2704  0.0  0.0 711448  7700 ?        Sl   Jul15   0:04 /usr/bin/cont
root        2705  0.0  0.1 711448  9104 ?        Sl   Jul15   0:04 /usr/bin/cont
65535       2742  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
65535       2749  0.0  0.0    972     4 ?        Ss   Jul15   0:00 /pause
root        2799  0.0  0.0 712856  8016 ?        Sl   Jul15   0:04 /usr/bin/cont
root        2840  0.1  0.3 735984 30244 ?        Ssl  Jul15   2:20 /storage-prov
root        2873  0.0  0.1 711448  9016 ?        Sl   Jul15   0:04 /usr/bin/cont
root        2893  0.2  0.5 750840 44156 ?        Ssl  Jul15   2:35 /coredns -con
root      180485  0.0  0.0   5904  2744 pts/1    Rs+  11:35   0:00 ps -aux
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$
````

</details>


## Tear down/clean-up

<details>
  <summary>Click to expand!</summary>
  

We saw we can do:
- `k3d cluster stop mycluster`,
- `minikube stop`: https://minikube.sigs.k8s.io/docs/commands/stop/

And both have delete

k3s not really something to stop as kubernetes as no VM, Docker layer managed by k3s.
Do not confuse with: https://rancher.com/docs/k3s/latest/en/installation/uninstall/

We can at least remove all resources

From  https://stackoverflow.com/questions/47128586/how-to-delete-all-resources-from-kubernetes-one-time

````
kubectl delete all --all -n {namespace} 
````

can we do?

````
kubectl delete all --all --all-namespaces
````

looking here: https://stackoverflow.com/questions/33509194/command-to-delete-all-pods-in-all-kubernetes-namespaces, yes


It will delete `kube-system` but those pod, deployment, will be recreated by Operator unlike `hello pod` one (we removed `hello deployment`)

````

Note if machine restart (minikube stop) and do `minikube start --nodes 2 -p multinode-demo --driver=docker` will "Restarting existing docker container for "multinode-demo"", if container still running because we just did a stop it will: "Updating the running docker "multinode-demo-m02" container ..."

 Docker which are cluster nodes keeps the same id

scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED        STATUS         PORTS
    NAMES
38a2b8271294   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   45 hours ago   Up 2 minutes   127.0.0.1:49162->22/tcp, 127.0.0.1:49161->2376/tcp, 127.0.0.1:49160->5000/tcp, 127.0.0.1:49159->8443/tcp, 127.0.0.1:49158->32443/tcp   multinode-demo-m02
2bc06cebf32e   gcr.io/k8s-minikube/kicbase:v0.0.30   "/usr/local/bin/entr…"   45 hours ago   Up 2 minutes   127.0.0.1:49157->22/tcp, 127.0.0.1:49156->2376/tcp, 127.0.0.1:49155->5000/tcp, 127.0.0.1:49154->8443/tcp, 127.0.0.1:49153->32443/tcp   multinode-demo

````



````
minikube stop ;  or machine reboot
minikube start --nodes 2 -p multinode-demo --driver=docker
sleee 5

kubectl apply -f deployment.yaml
kubectl get pods --all-namespaces

kubectl delete all --all --all-namespaces
kubectl get pods --all-namespaces
````

Output is

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl apply -f deployment.yaml
deployment.apps/hello created
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                     READY   STATUS              RESTARTS        AGE
default       hello-99b498c79-5cchb                    1/1     Running             0               30s
default       hello-99b498c79-8wdln                    0/1     ContainerCreating   0               30s
default       hello-99b498c79-c9v7p                    1/1     Running             0               30s
default       hello-99b498c79-df2g9                    1/1     Running             0               30s
default       hello-99b498c79-gbm9k                    1/1     Running             0               30s
default       hello-99b498c79-hzrdt                    1/1     Running             0               30s
default       hello-99b498c79-jfcqz                    1/1     Running             0               30s
default       hello-99b498c79-jzv55                    1/1     Running             0               30s
default       hello-99b498c79-njghp                    1/1     Running             0               30s
default       hello-99b498c79-r44ff                    1/1     Running             0               30s
default       hello-99b498c79-r4kdq                    1/1     Running             0               30s
default       hello-99b498c79-r4kmc                    1/1     Running             0               30s
default       hello-99b498c79-s4s6s                    1/1     Running             0               30s
default       hello-99b498c79-t6vgk                    1/1     Running             0               30s
default       hello-99b498c79-wr8tx                    1/1     Running             0               30s
kube-system   coredns-64897985d-89p5s                  1/1     Running             1 (8m26s ago)   14m
kube-system   etcd-multinode-demo                      1/1     Running             3 (8m26s ago)   26m
kube-system   kindnet-d6nw9                            1/1     Running             1 (8m26s ago)   14m
kube-system   kindnet-p8xl8                            0/1     CrashLoopBackOff    12 (64s ago)    14m
kube-system   kube-apiserver-multinode-demo            1/1     Running             3 (8m26s ago)   26m
kube-system   kube-controller-manager-multinode-demo   1/1     Running             3 (8m26s ago)   26m
kube-system   kube-proxy-hxvk8                         0/1     CrashLoopBackOff    12 (2m5s ago)   14m
kube-system   kube-proxy-w7wqx                         1/1     Running             1 (8m26s ago)   14m
kube-system   kube-scheduler-multinode-demo            1/1     Running             3 (8m26s ago)   26m
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl delete all --all --all-namespaces
pod "hello-99b498c79-5cchb" deleted
pod "hello-99b498c79-8wdln" deleted
pod "hello-99b498c79-c9v7p" deleted
pod "hello-99b498c79-df2g9" deleted
pod "hello-99b498c79-gbm9k" deleted
pod "hello-99b498c79-hzrdt" deleted
pod "hello-99b498c79-jfcqz" deleted
pod "hello-99b498c79-jzv55" deleted
pod "hello-99b498c79-njghp" deleted
pod "hello-99b498c79-r44ff" deleted
pod "hello-99b498c79-r4kdq" deleted
pod "hello-99b498c79-r4kmc" deleted
pod "hello-99b498c79-s4s6s" deleted
pod "hello-99b498c79-t6vgk" deleted
pod "hello-99b498c79-wr8tx" deleted
pod "coredns-64897985d-89p5s" deleted
pod "etcd-multinode-demo" deleted
pod "kindnet-d6nw9" deleted
pod "kindnet-p8xl8" deleted
pod "kube-apiserver-multinode-demo" deleted
pod "kube-controller-manager-multinode-demo" deleted
pod "kube-proxy-hxvk8" deleted
pod "kube-proxy-w7wqx" deleted
pod "kube-scheduler-multinode-demo" deleted
service "kubernetes" deleted
service "kube-dns" deleted
daemonset.apps "kindnet" deleted
daemonset.apps "kube-proxy" deleted
deployment.apps "hello" deleted
deployment.apps "coredns" deleted

scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                     READY   STATUS    RESTARTS       AGE
kube-system   etcd-multinode-demo                      1/1     Running   3 (9m6s ago)   32s
kube-system   kube-apiserver-multinode-demo            1/1     Running   3 (9m6s ago)   32s
kube-system   kube-controller-manager-multinode-demo   1/1     Running   3 (9m6s ago)   32s
kube-system   kube-scheduler-multinode-demo            1/1     Running   3 (9m6s ago)   32s

scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ kubectl describe po kube-apiserver-multinode-demo -n kube-system | grep -A 15 Events:
Events:
  Type    Reason          Age   From     Message
  ----    ------          ----  ----     -------
  Normal  SandboxChanged  42m   kubelet  Pod sandbox changed, it will be killed and re-created.
  Normal  Pulled          42m   kubelet  Container image "k8s.gcr.io/kube-apiserver:v1.23.3" already present on machine
  Normal  Created         42m   kubelet  Created container kube-apiserver
  Normal  Started         42m   kubelet  Started container kube-apiserver
  Normal  SandboxChanged  16m   kubelet  Pod sandbox changed, it will be killed and re-created.
  Normal  Pulled          16m   kubelet  Container image "k8s.gcr.io/kube-apiserver:v1.23.3" already present on machine
  Normal  Created         16m   kubelet  Created container kube-apiserver
  Normal  Started         16m   kubelet  Started container kube-apiserver
  Normal  SandboxChanged  10m   kubelet  Pod sandbox changed, it will be killed and re-created.
  Normal  Pulled          10m   kubelet  Container image "k8s.gcr.io/kube-apiserver:v1.23.3" already present on machine
  Normal  Created         10m   kubelet  Created container kube-apiserver
  Normal  Started         10m   kubelet  Started container kube-apiserver
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$

````

We can see kube-sytem pod were re-created by internal controller.
We keep restart histort as pod name is the same. Thus restart > age.

</details>
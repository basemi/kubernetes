# Installing kubernetes on Debian 11

[Questa guida in linguaggio Italiano](./create_k8s_debian_cluster_it.md)

This guide is intended to help you to install k8s on debian 11 bullseye.

This guide was written starting from the official Kubernetes pages [Installation](https://kubernetes.io/docs/setup/). There is more than one method to install k8s, this method is based on *kubeadmnin*.

<img src="kubeadm-stacked-color.png"  width="200" height="200">.

## Prerequisites

  * Machines with Debian 11 installed;
  * At least 2GB of ram per host;
  * 2 CPU or VCPU per host;
  * Full network connectivity between hosts;
  * Unique *Hostname*, *mac address* e *product uuid* per host.
    * To check the mac addresses you can use the command **ip link** or **ifconfig -a**;
    * The *product uuid* is obtained with the command **sudo cat /sys/class/dmi/id/product_uuid**;
  * Swap must be disabled for each host (**swapoff -a**). It should be disabled directly on the */etc/fstab* file to avoid having to execute **swapoff -a** each time the host is restarted;
  * Installation of the *container runtime*.


### Checking the ports

Specific ports must be enabled for hosts in the cluster. For this argument we need to distinguish between **master** node and **slave**/**worker** nodes.

#### Nodo Master

| Protocol | Direction | Ports    | Description                |
|----------|-----------|----------|----------------------------|
| TCP      | ingress   | 6443     | Kubernetes API server      |
| TCP      | ingress   | 2379-2380| **etcd** server client API |
| TCP      | ingress   | 10250    | **Kubelet** API            |
| TCP      | ingress   | 10259    | **kube-scheduler**         |
| TCP      | ingress   | 10257    | **kube-controller-manager**|

#### Nodi slave o worker

| Protocol | Direction | Ports      | Description          |
|----------|-----------|------------|----------------------|
| TCP      | ingress   | 10250      | **Kubelet** API      |
| TCP      | ingress   | 30000-32767| **NodePort** services|

## Container runtime

A *container runtime* must be installed on each node. With version 1.27 of kubernetes, you can only use runtimes that are **Container Runtime Interface (CRI)** compliant.

There are several runtimes that can be used for this purpose. **Docker** could be used but it would not be sufficient, as **docker** is not compliant with the **CRI** standard and therefore will need to be supported by an additional component that is [cri-dockerd](https://github.com/Mirantis/cri-dockerd). An alternative to **docker** will be used [containerd](https://github.com/containerd/containerd).

### Network configuration

We execute the command:

    $ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter

    # sysctl params required by setup, params persist across reboots
    $ cat <<EOF| sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

    # Apply sysctl params without reboot
    $ sudo sysctl --system

We need to verify that the **br_netfilter** and **overlay** kernel modules are loaded:

    $ lsmod | grep br_netfilter
    $ lsmod | grep overlay

Verify that **net.bridge.bridge-nf-call-iptables** and **net.bridge.bridge-nf-call-ip6tables**, **net.ipv4.ip_forward** are set to 1:

    $ sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

Once this is done, for each node you need to add the *ip* and *dns* table of each node in the cluster to the */etc/hosts* file. Alternatively you can use a *dns* server like **bind**.

### Installing containderd

In debian it can be installed via *apt* but using the version released by *docker*. The version shipped with debian is not compatible with the latest version of *kubernetes*.

Let's install **curl** and **apt** support for the https protocol:

    $ sudo apt-get update
    $ sudo apt-get install -y apt-transport-https ca-certificates curl

Let's add the repository for docker and containerd. For versions from 11 onwards the */etc/apt/keyrings* folder does not exist and must be created before executing the instructions to add the repository:

    $ sudo install -m 0755 -d /etc/apt/keyrings
    $ curl -fsSL https://download.docker.com/linux/debian/gpg| sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    $ sudo chmod a+r /etc/apt/keyrings/docker.gpg

    $ echo \
      "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
      "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable"| \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    $ sudo apt-get update
    $ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

Let's apply the default settings:

    $ sudo containerd config default| sudo tee /etc/containerd/config.toml >/dev/null 2>&1

We need to change the configuration to use the *cgroup* driver with **systemd** instead of cgroupfs.

Edit the */etc/containerd/config.toml* file so that after the line:

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]

there is **SystemdCgroup = true**.

We start **containerd** and enable it to start on boot:

    $ sudo systemctl restart containerd
    $ sudo systemctl enable containerd

## Installing kubernetes

Let's enable the *kubernetes* repository on debian:

Adding the new repository:

    $ sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    $ sudo echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main"| sudo tee /etc/apt/sources.list.d/kubernetes.list

Installiamo **kubeadm**, **kubelet** e **kubectl** e ne blocchiamo le versioni:

    $ sudo apt-get update
    $ sudo apt-get install -y kubelet kubeadm kubectl
    $ sudo apt-mark hold kubelet kubeadm kubectl

## Activating the cluster

The installation of the cluster is done with the command:

    $ sudo kubeadmin init

If all went well, we'll get something like this:

> Your Kubernetes control-plane has initialized successfully!
>
> To start using your cluster, you need to run the following as a regular user:
>
    $ mkdir -p $HOME/.kube
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
>
> Alternatively, if you are the root user, you can run:
>
> export KUBECONFIG=/etc/kubernetes/admin.conf

> You should now deploy a pod network to the cluster.
> Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
>  https://kubernetes.io/docs/concepts/cluster-administration/addons/
>
> You can now join any number of control-plane nodes by copying certificate authorities
> and service account keys on each node and then running the following as root:
>
>  kubeadm join master:6443 --token wb4p8t.kethcu6y2a7dz75n \
>	--discovery-token-ca-cert-hash sha256:a870710a2002c3b793e54045ab52095226934e854d613e3a80bf6630ad9d01c9 \
> 	--control-plane
>
> Then you can join any number of worker nodes by running the following on each as root:
>
> kubeadm join master:6443 --token wb4p8t.kethcu6y2a7dz75n \
> 	--discovery-token-ca-cert-hash sha256:a870710a2002c3b793e54045ab52095226934e854d613e3a80bf6630ad9d01c9

### Adding the nodes to the cluster

To interact with the cluster we need to configure the user who will execute the commands with **kubectl**. Therefore, as user **not** *root* we execute the commands:

    $ mkdir -p $HOME/.kube
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config

To get back or regenerate the token, from the *master* node, run the command:

    $ sudo kubeadm token create --print-join-command

From the *slave* nodes we execute the command:

    # xxxxx depends on what the generate token command returns
    $ sudo kubeadm join master:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx

After adding all nodes to the cluster, we run the command:

    $ kubectl get nodes

We will get a response similar to this:

    user@master:~$ kubectl get nodes
    NAME     STATUS     ROLES           AGE    VERSION
    master   NotReady   control-plane   40h    v1.27.1
    slave1   NotReady   <none>          175m   v1.27.1
    slave2   NotReady   <none>          20s    v1.27.0

Status is **NotReady** due to missing overlay network for **pods**.

The *overlay* network is a network that sits "above" the nodes of the cluseter and is used to make the nodes communicate with each other.

### Enabling the overlay network for pods

For the pods to interact with each other you need to install a *POD network addon*. Such an addon does nothing but install a network overlay for pods. There are several, at the link [Network plugin](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy) you have a list of possible choices.

For simplicity, the possible choices fall on:

  * [Calico](https://www.tigera.io/project-calico/);
  * [Flannel](https://github.com/flannel-io/flannel);
  * [Weave net](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/).

In our case we will use **Weave net**. From the master node and as a "normal" user:

    $ kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

We would have as a result:

    $ user@master:~$ kubectl apply -f "https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml"
    serviceaccount/weave-net created
    clusterrole.rbac.authorization.k8s.io/weave-net created
    clusterrolebinding.rbac.authorization.k8s.io/weave-net created
    role.rbac.authorization.k8s.io/weave-net created
    rolebinding.rbac.authorization.k8s.io/weave-net created
    daemonset.apps/weave-net created

Running the `kubectl get nodes` command will give:

    user@master:~$ kubectl get nodes
    NAME     STATUS   ROLES           AGE    VERSION
    master   Ready    control-plane   40h    v1.27.1
    slave1   Ready    <none>          3h7m   v1.27.1
    slave2   Ready    <none>          12m    v1.27.0

All nodes are **Ready**.

### Testing the cluster

To test the cluster, let's create a deploy:

    $ kubectl create deployment --image nginx my-nginx --replicas 3

We used the **nginx** image with 3 replicas. The name of this *deployment* is **my-nginx**.

To check the status of the *deployment* creation:

    $ user@master:~$ kubectl get pods
    NAME                      READY   STATUS              RESTARTS   AGE
    my-nginx-b8dd4cd6-fpwpz   0/1     ContainerCreating   0          40s
    my-nginx-b8dd4cd6-qfxk7   0/1     ContainerCreating   0          40s
    my-nginx-b8dd4cd6-v24hl   0/1     ContainerCreating   0          40s

Containers are still being created.

After a few minutes:

    $ user@master:~$ kubectl get pods
    $ NAME                      READY   STATUS    RESTARTS   AGE
    $ my-nginx-b8dd4cd6-fpwpz   1/1     Running   0          3m54s
    $ my-nginx-b8dd4cd6-qfxk7   1/1     Running   0          3m54s
    $ my-nginx-b8dd4cd6-v24hl   1/1     Running   0          3m54s

We see that the three replicas have been created.

Another command that shows us that the three replicas are up:

    $ user@master:~$ kubectl get deployment
    NAME       READY   UP-TO-DATE   AVAILABLE   AGE
    my-nginx   3/3     3            3           9m29s

If we want to have the details of the pods created:

    $ user@master:~$ kubectl describe pod my-nginx
    Name:             my-nginx-b8dd4cd6-fpwpz
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             slave1/192.168.0.176
    Start Time:       Tue, 18 Apr 2023 13:36:01 +0000
    Labels:           app=my-nginx
                      pod-template-hash=b8dd4cd6
    Annotations:      <none>
    Status:           Running
    IP:               10.40.0.3
    IPs:
      IP:           10.40.0.3
    Controlled By:  ReplicaSet/my-nginx-b8dd4cd6
    Containers:
      nginx:
        Container ID:   containerd://784852248f6e1c0d56ccaaf40fc6217aefea34f29afe2f9359ed694088119dfc
        Image:          nginx
        Image ID:       docker.io/library/nginx@sha256:63b44e8ddb83d5dd8020327c1f40436e37a6fffd3ef2498a6204df23be6e7e94
    ...
    ...

We see that the ip of the pod is **10.40.0.3**, we can test the nginx http server:

    $ user@master:~$ curl 10.40.0.3
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>

We see that **nginx** answered correctly.

Optionally, we can expose **my-nginx** outside the pod network:

    $ kubectl expose deployment my-nginx --name=nginx-http --type NodePort --port 80 --target-port 80
    service/nginx-http exposed
    $ kubectl describe svc nginx-http
    Name:                     nginx-http
    Namespace:                default
    Labels:                   app=my-nginx
    Annotations:              <none>
    Selector:                 app=my-nginx
    Type:                     NodePort
    IP Family Policy:         SingleStack
    IP Families:              IPv4
    IP:                       10.101.79.21
    IPs:                      10.101.79.21
    Port:                     <unset>  80/TCP
    TargetPort:               80/TCP
    NodePort:                 <unset>  31561/TCP
    Endpoints:                10.40.0.3:80,10.40.0.4:80,10.40.0.5:80
    Session Affinity:         None
    External Traffic Policy:  Cluster
    Events:                   <none>

We can query **my-nginx** on one of the nodes using port **31561**.

    $ user@master:~$ curl master:31561
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>
    
    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>
    
    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>

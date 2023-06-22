#Splitting the traffic between two services using Traefik Ingress Proxy:

![Traefik Split](https://raw.githubusercontent.com/shkatara/traefik-split-applications/main/trivago_traefik_split.gif)


##To be able to install the cluster, I am bootstraping a cluster using kubeadm version 1.27.3. I am following on a single master node that also works as the worker. Consider the following pre-requisistes before installing the cluster. 
======
1. Install Kubeadm, kubelet and kubectl from github / kubernetes website. 
2. Install a container runtime. For the simplicity, I am using docker from Docker Inc. 
3. I will be disabling firewalld because we are using calico. These rules may interfere with the rules added by Calico and result in some strange behaviour. 
4. If you are using NetworkManager, it has to be configured before attempting to use calico networking. 
5. Also, install helm on your machine. 

##System Information on which the setup is performed:

```bash
[root@kubesys ~]# cat /etc/os-release
NAME="CentOS Linux"
VERSION="8"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="8"
PLATFORM_ID="platform:el8"
PRETTY_NAME="CentOS Linux 8"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:8"
HOME_URL="https://centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"
CENTOS_MANTISBT_PROJECT="CentOS-8"
CENTOS_MANTISBT_PROJECT_VERSION="8"
```

```bash
[root@osboxes ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           3735        1782         141          10        1812        1723
Swap:             0           0           0
```

NOTE: Write the following in /etc/NetworkManager/conf.d/calico.conf. As NetworkManager manipulates routing for network interfaces in the default network namespace where calico veth pair's are created, we do not want networkmanager to interfere with these interface's routing. 

```bash
[root@kubesys ~]# vi /etc/NetworkManager/conf.d/calico.conf

[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
```
##Install the cluster using kubeadm, do the following
=====

```bash
[root@kubesys ~]#  kubeadm init --pod-network-cidr 10.244.0.0/16
```
##Configure your root user to be able to connect with kuberneres cluster
=====

```bash
[root@kubesys ~]#  cp /etc/kubernetes/admin.conf ~/.kube/config
[root@kubesys ~]#  kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
##Install calico using the Calico Operator
=====
```bash
[root@kubesys ~]#  kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
[root@kubesys ~]#  kubectl create -f custom-resources.yml
```
##Deploy Traefik Ingress Controller

```bash
[root@kubesys ~]#  helm repo add traefik https://helm.traefik.io/traefik
[root@kubesys ~]#  helm repo update
[root@kubesys ~]#  helm install traefik traefik/traefik
NAME: traefik
LAST DEPLOYED: Thu Jun 22 19:21:31 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

##Deploy the test applications. 
The application consists of 1 namespace manifest, under which 2 different deployments will be created. For each deployment, there will be respective services and also will be the ingress resources that will be created to expose the application to the outside world. Later on, we will see how we can expose the applications using one single ingress to split the traffic on other nodes. 

```bash
[root@kubesys ~]#  kubectl create -f trivago/app-different-versions/namespace.yml create
[root@kubesys ~]#  kubectl create -f trivago/app-different-versions/version1-deploy.yml create
[root@kubesys ~]#  kubectl create -f trivago/app-different-versions/version2-deploy.yml create
[root@kubesys ~]#  kubectl get all,ingress -n trivago
```

##Test the application Ingresses.
As we have deployed traefik, by default in the simplest configuration, it creates a LoadBalancer type of service and exposes ports for port 80 and 443 on the node. So, when we are routing traffic to the ingress using URLs, we need to append the port number of the service for HTTP. 

NOTE: As there is no DNS for us to talk to the applications, make a temporary resolution in your /etc/hosts file like the following:

192.168.1.2 version1.trivago.apps.com version2.trivago.apps.com 

```bash
[root@kubesys ~]#  export web_port=$(kubectl  get svc traefik -o jsonpath='{.spec.ports[0].nodePort}')
[root@kubesys ~]#  curl version1.trivago.apps.com:$web_port
I am version v1!
[root@kubesys ~]#  curl version2.trivago.apps.com:$web_port
I am version v2!
```

#Splitting Traffic between two application versions.
We will be using Traefik Proxy to split traffic between two different applications. This lets us split traffic without having to go through the overhead of istio or any other service mesh. 

There are two resources we will deploy to work with splitting the traffic between two services with Traefik:
1. TraefikService: TraefikService is an API resource in Kubernetes that is an implementation of traefik service. With traefik service resource, we can implement load balancing to balance the requests between multiple services based on weights. 
2. IngressRoute: The Ingress Route is incharge of connecting incoming requests to the services that can handle them, usually the traefiksesrvice. The IngressRoute can also  send traffic to Middleware API resource to update the request or act before forwarding the request to the service. IngressRoutes consists of routes that once are matched, we can transform or forward the request to one or more services based on http headers as well, or just plain simple round robin or weighted traffic split. 


Let us create the given two resources and we will then check for the application if we are able to split the traffic or not. 

```bash
[root@kubesys ~]# kubectl -f traefik-ingress-controller/traefikservice.yml create
[root@kubesys ~]# kubectl -f traefik-ingress-controller/ingressroute.yml create 
```

Let us test the traffic splitting between two applications. We will send 100 requests and it should be spliiting 70% to the first application, and the rest 30% should be sent to the second application. This will prove that our goal is achieved to split the traffic

```bash
[root@kubesys ~]# for i in {1..100}; do curl version1.trivago.apps.com:web_port 2>/dev/null; done | grep v1 | wc -l
70
[root@kubesys ~]# for i in {1..100}; do curl version1.trivago.apps.com:web_port 2>/dev/null; done | grep v2 | wc -l
30
```

As we can see, the traffic out of 100 requests, 70 are being sent to first application and the rest 30% is being sent to the other application. Hence, we know the setup and the infrastructure is working as expected.

##To be able to install the cluster, I am bootstraping a cluster using kubeadm version 1.27.3. I am following on a single master node that also works as the worker. Consider the following pre-requisistes before installing the cluster. 
======
1. Install Kubeadm, kubelet and kubectl from github / kubernetes website. 
2. Install a container runtime. For the simplicity, I am using docker from Docker Inc. 
3. I will be disabling firewalld because we are using calico. These rules may interfere with the rules added by Calico and result in some strange behaviour. 
4. If you are using NetworkManager, it has to be configured before attempting to use calico networking. 
5. Also, install helm on your machine. 


Write the following in /etc/NetworkManager/conf.d/calico.conf. As NetworkManager manipulates routing for network interfaces in the default network namespace where calico veth pair's are created, we do not want networkmanager to interfere with these interface's routing. 

[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali

##Install the cluster using kubeadm, do the following
=====

[root@kubesys ~]#  kubeadm init --pod-network-cidr 10.244.0.0/16

##Configure your root user to be able to connect with kuberneres cluster
=====

[root@kubesys ~]#  cp /etc/kubernetes/admin.conf ~/.kube/config
[root@kubesys ~]#  kubectl taint nodes --all node-role.kubernetes.io/control-plane-

##Install calico using the Calico Operator
=====

[root@kubesys ~]#  kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
[root@kubesys ~]#  kubectl create -f custom-resources.yml

##Deploy Traefik Ingress Controller

[root@kubesys ~]#  helm repo add traefik https://helm.traefik.io/traefik
[root@kubesys ~]#  helm repo update
[root@kubesys ~]#  helm install traefik traefik/traefik
NAME: traefik
LAST DEPLOYED: Thu Jun 22 19:21:31 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None

##Deploy the test applications. 
The application consists of 1 namespace manifest, under which 2 different deployments will be created. For each deployment, there will be respective services and also will be the ingress resources that will be created to expose the application to the outside world. Later on, we will see how we can expose the applications using one single ingress to split the traffic on other nodes. 


[root@kubesys ~]#  kubectl create -f trivago/app-different-versions/namespace.yml create
[root@kubesys ~]#  kubectl create -f trivago/app-different-versions/version1-deploy.yml create
[root@kubesys ~]#  kubectl create -f trivago/app-different-versions/version2-deploy.yml create
[root@kubesys ~]#  kubectl get all,ingress -n trivago

##Test the application Ingresses.
As we have deployed traefik, by default in the simplest configuration, it creates a LoadBalancer type of service and exposes ports for port 80 and 443 on the node. So, when we are routing traffic to the ingress using URLs, we need to append the port number of the service for HTTP. 

NOTE: As there is no DNS for us to talk to the applications, make a temporary resolution in your /etc/hosts file like the following:
```
192.168.1.2 version1.trivago.apps.com version2.trivago.apps.com 
```

```bash
[root@kubesys ~]#  export web_port=$(kubectl  get svc traefik -o jsonpath='{.spec.ports[0].nodePort}')
[root@kubesys ~]#  curl version1.trivago.apps.com:$web_port
I am version v1!
[root@kubesys ~]#  curl version2.trivago.apps.com:$web_port
I am version v2!
```

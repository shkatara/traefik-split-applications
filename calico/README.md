1. Install the kubernetes cluster using kubeadm.

kubeadm init --pod-network-cide 10.244.0.0/16

1. Install the operator once the nodes are ready:

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

2. Create the manifest for creating calico apiserver

kubectl -f custom-resources.yaml create

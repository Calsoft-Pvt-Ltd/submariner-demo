# Connecting multiple kubernetes clusters

This respository explains how we can connect multiple kubernetes clusters using the submariner tool https://submariner.io/

[This repository has been tested on VM's that were created in AWS cloud]

## Management VM
Create a management VM from where we can run all the commands, please make sure this VM has following tools installed

1. ansible https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-22-04 

2. kubectl https://kubernetes.io/docs/tasks/tools/

3. subctl  
curl -Ls https://get.submariner.io | bash
export PATH=$PATH:~/.local/bin
echo export PATH=\$PATH:~/.local/bin >> ~/.profile


## Source Code

The playbook directory has the ansible script to create kubernetes cluster

### Configuration

The inventory.yaml should be edited as per your VM host IPs and security credentials [pem file] This kubernetes cluster makes use of flannel CNI plugin hence the Pod_subnet range should always be 10.244.x.x/16 and ensure this range does not overlap with anyother cluster

### Test Pods
The test folder contains the deployment and service configuration for the pods that are installed in these cluster and are used to test the communication

nginx-demo.yaml -> This creates a nginx pod in one of the cluster with cluster IP configuration

netshoot.yaml -> This is deployed on another cluster to test the ping/curl to the nginx pod which is another cluster

## Kubernetes clusters

Following is the list of kubernetes clusters we can create

1. Cluster A - 2 VM [one master and one worker] 
2. Cluster B - 2 VM [one master and one worker] also acts as sumbariner broker

### Create Clusters

Clone the repository in the management VM
Create a total of 4 VM's for 2 clusters and record their IPs, Make sure the worker node of each worker has a public IP and can be accessed from outside; also make sure the Security group setting allows all kind of inbound/outbound traffic to all these 4 VM's

Modify the inventory.yaml with the host IP address recorded above for VM's and security credentials[pem file] and execute in following order for each cluster

1. Update dependencies

ansible-playbook -i inventory.yaml kube-dependencies.yaml

2. Create master node

ansible-playbook -i inventory.yaml master.yaml

2. Create worker nodes

ansible-playbook -i inventory.yaml workers.yaml


# Install Submariner and connect two kuberentes clusters 

Install subctl on both the cluster master node, above all commands are executed on respective clusters master node

## Deploy Broker and join clusters

We are making cluster B as broker 

### Login to Cluster B

subctl deploy-broker --kubeconfig .kube/config

## Join Cluster B to the Broker

subctl join --kubeconfig .kube/config broker-info.subm --clusterid clusterb

Note: broker-info.subm is the file auto generated from above command of deploy broker

### Login to Cluster A

## Join Cluster A to the Broker

Note: copy the broker-info.subm from cluster B to cluster A

subctl join --kubeconfig .kube/config broker-info.subm --clusterid clustera

## Login to Cluster B

### Diagnose cluster 

subctl diagnose all

### Check cluster 

kubectl get clusters --all-namespaces

### Describe clusters

kubectl describe cluster --all-namespaces

### Check Endpoint 

kubectl get endpoint --all-namespaces

### Create nginx Deployment with its service with cluster IP
Note: copy the test/nginx-demo.yaml on the cluster B master node 

kubectl apply -f nginx-demo.yaml

### Check nginx service info 

kubectl get svc -l app=nginx-demo

Note: Note down the cluster IP of nginx service

## Login to Cluster A

### Create netshoot deployment 
Note: copy the test/netshoot.yaml on the cluster A master node 

kubectl apply -f netshoot.yaml

### Get netshoot pod name

kubectl get pods -l app=netshoot
Note: Record the name of the pod

### Curl Nginx in Cluster A from Cluster B netshoot pod

kubectl exec [netshoot pod name] -- curl [ClusterIP of Nginx-demo]


# Create a Kubeadm cluster

The following guide is based on the documentation from Calico OSS [self-managed GCE k8s installation](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-public-cloud/gce) for provisioning compute resources.

**Goal:** Create a self-managed K8s cluster.

## Prerequisite Tasks

### Utility 

- Ensure you have installed the [Cloud SDK](https://cloud.google.com/sdk/docs/install)

- Ensure you set up default gcloud settings using one of the following methods:
   
   Using gcloud init, if you want to be walked through setting defaults.
   Using gcloud config, to individually set your project ID, zone, and region.

   ```bash
   gcloud init                                                                    
   Welcome! This command will take you through the configuration of gcloud.
   ```

   Then be sure to authorize gcloud to access the Cloud Platform with your Google user credentials:

   ```bash
   gcloud auth login
   ```

   Next set a default compute region and compute zone:

   ```bash
   gcloud config set compute/region us-east1
   ```

   Set a default compute zone:

   ```bash
   gcloud config set compute/zone us-east1-b
   ```


### Provisioning networking

1. Create the VPC

   In this section a dedicated [Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC) network will be setup to host the Kubernetes cluster.

   Create the `calicocloud-vpc` custom VPC network:

   ```bash
   gcloud compute networks create calicocloud-vpc --subnet-mode custom
   ```


2. Create the subnet

   A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

   Create the `k8s-nodes` subnet in the `calicocloud-vpc` VPC network:

   ```bash
   gcloud compute networks subnets create k8s-nodes \
   --network calicocloud-vpc \
   --range 10.240.0.0/24
   ```

   > The `10.240.0.0/24` IP address range can host up to 254 compute instances.


3. Create the firewall rules

   a. Create a firewall rule that allows internal communication across all protocols:

   ```bash
   gcloud compute firewall-rules create calicocloud-vpc-allow-internal \
   --allow tcp,udp,icmp,ipip \
   --network calicocloud-vpc \
   --source-ranges 10.240.0.0/24,192.168.0.0/16
   ```
   > Calico overlay also require the 'ipip' protocol, which is out of scope for this workshop.

   b. Create a firewall rule that allows external SSH, ICMP, and HTTPS:

   ```bash
   gcloud compute firewall-rules create calicocloud-vpc-allow-external \
   --allow tcp,udp,icmp \
   --network calicocloud-vpc \
   --source-ranges 0.0.0.0/0
   ```

   > An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

   c. List the firewall rules in the `calicocloud-vpc` VPC network:

   ```bash
   gcloud compute firewall-rules list --filter="network:calicocloud-vpc"
   ```

   > output

   ```bash
   NAME                            NETWORK          DIRECTION  PRIORITY  ALLOW              DENY  DISABLED
   calicocloud-vpc-allow-external  calicocloud-vpc  INGRESS    1000      tcp,udp,icmp             False
   calicocloud-vpc-allow-internal  calicocloud-vpc  INGRESS    1000      tcp,udp,icmp,ipip        False
   ```

### Provisioning compute instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04. We will have 1 master node + 2 worker node.

1. Create one controller node

   ```bash
   gcloud compute instances create master-node \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-4 \
    --private-network-ip 10.240.0.11 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-nodes \
    --zone us-east1-b \
    --tags calicocloud,master

   ```

2. Create two compute instances which will host the Kubernetes worker nodes:   

   ```bash
   for i in 0 1; do
   gcloud compute instances create worker-node${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-4 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-nodes \
    --zone us-east1-b \
    --tags calicocloud,worker
   done
   ```

3. Verify the instances are running as desired. 

   ```bash
   gcloud compute instances list --filter="tags.items=calicocloud"
   ```

   > Output 
   ```bash
   NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
   master-node   us-east1-b  e2-standard-4               10.240.0.11  XX.XX.XX.XX    RUNNING
   worker-node0  us-east1-b  e2-standard-4               10.240.0.20  XX.XX.XX.XX    RUNNING
   worker-node1  us-east1-b  e2-standard-4               10.240.0.21  XX.XX.XX.XX    RUNNING
   ```

4. Configuring SSH access to each instance.

   SSH will be used to configure the master and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as described in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

   Test SSH access to the `master-node` compute instances as example:

   ```bash
   gcloud compute ssh master-node
   ```

   If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

   
   > Output 
   ```bash
   WARNING: The public SSH key file for gcloud does not exist.
   WARNING: The private SSH key file for gcloud does not exist.
   WARNING: You do not have an SSH key for gcloud.
   WARNING: SSH keygen will be executed to generate a key.
   Generating public/private rsa key pair.
   Enter passphrase (empty for no passphrase):
   Enter same passphrase again:
   ```

   At this point the generated SSH keys will be uploaded and stored in your project:

   ```bash
   Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
   Updating project ssh metadata...done.
   Waiting for SSH key to propagate.
   ```

   After the SSH keys have been updated you'll be logged into the `master-node` instance:

   ```bash
   Welcome to Ubuntu 18.04.6 LTS (GNU/Linux 5.4.0-1053-gcp x86_64)
   ```

   > Type `exit` at the prompt to exit the `master-node` compute instance, and you should be able to ssh other nodes with same key.   


## Steps

1. Install containerd & K8S components on the master VM and each worker VM. On each VM run:

   ```bash
   sudo modprobe overlay && sudo modprobe br_netfilter
   ```
   
   ```bash
   cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
   overlay
   br_netfilter
   EOF
   ```

   ```bash
   cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
   net.bridge.bridge-nf-call-iptables  = 1
   net.ipv4.ip_forward                 = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   EOF
   ```

   ```bash
   sudo sysctl --system && sudo apt-get update -y && sudo apt-get install -y containerd
   ```

   ```bash
   sudo mkdir -p /etc/containerd
   sudo containerd config default | sudo tee /etc/containerd/config.toml
   sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
   ```

   ```bash
   sudo systemctl restart containerd
   ```

   ```bash
   sudo apt install -y apt-transport-https curl
   
   sudo sh -c "echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' >> /etc/apt/sources.list.d/kubernetes.list"
  
   sudo sh -c "curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -"
   
   sudo apt-get update && sudo apt-get install -y kubeadm=1.22.4-00 kubelet=1.22.4-00 kubectl=1.22.4-00
   
   sudo apt-mark hold kubelet kubeadm kubectl
   ```
 
2. Initiate the master node with pod CIDR config. On master node run: 
 
   ```bash
   sudo kubeadm init --kubernetes-version 1.22.4 --pod-network-cidr 192.168.0.0/16 --apiserver-cert-extra-sans=127.0.0.1
   ```

   > output include the token for worker node to join to the cluster, you can save them in txt if you want to use it later.  

   ```bash
   Then you can join any number of worker nodes by running the following on each as root:

   ##below is the token for worker node.
   kubeadm join 10.240.0.11:6443 --token XXXXXXXXXXX \
	--discovery-token-ca-cert-hash sha256:2a459afXXXXXXXXXXXa30a9cacb42XXXXXXXXXXXa761c1fbXXXXXXXXXXX
   ```

3.  set up kubectl for the ubuntu user. On master node run:

   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

4. The final step is joining your worker node to master node. Run this on each worker, prepending sudo to run it as root. It will look something like this:
   
   ```bash
   sudo kubeadm join 10.240.0.11:6443 --token XXXXXXXXXXX \
   --discovery-token-ca-cert-hash sha256:2a459afXXXXXXXXXXXa30a9cacb42XXXXXXXXXXXa761c1fbXXXXXXXXXXX
   ```

5. On the controller, verify that all nodes have joined.

   ```bash
   kubectl get nodes
   ```

   >Output is 
   ```bash
   NAME           STATUS     ROLES                  AGE     VERSION
   master-node    NotReady   control-plane,master   2m20s   v1.22.4
   worker-node0   NotReady   <none>                 92s     v1.22.4
   worker-node1   NotReady   <none>                 13s     v1.22.4
   ```

6. On the controller, install calico OSS and then we can join this cluster to calico cloud as managed cluster. 

   ```bash
   kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
   kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml

   
   #Confirm the nodes are ready
   kubectl get nodes
   ```

   >Output is  
   ```bash
   NAME           STATUS   ROLES                  AGE   VERSION
   master-node    Ready    control-plane,master   4m19s   v1.22.4
   worker-node0   Ready    <none>                 3m31s   v1.22.4
   worker-node1   Ready    <none>                 2m12s   v1.22.4
   ```

7. Remove master taint in order to allow kubernetes to schedule pods on the master node.

   ```bash
   kubectl taint nodes --all node-role.kubernetes.io/master-
   ```

8. Download this repo into your environment:

   ```bash

   git clone https://github.com/tigera-solutions/calicocloud-kaas-workshop-demo.git

   cd calicocloud-kaas-workshop-demo
   ```   
--- 
## Next steps

You should now have a Kubernetes cluster running with 3 nodes. The Control Plane services which manage the Kubernetes cluster such as scheduling, API access, configuration data store and object controllers are all provided as services to the nodes.
<br>    


[:arrow_right: Join the cluster to Calico Cloud](./joining-calico-cloud.md)

[:leftwards_arrow_with_hook: README.md - STEP 1 - Create a compatible k8s cluster](../README.md#step-1---create-a-compatible-k8s-cluster)
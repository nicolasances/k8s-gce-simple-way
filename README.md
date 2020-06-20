This is a walkthough on how to create a K8S cluster on Google Compute Engine instances (not using GKE), how to **configure NGINX as an Ingress Controller without Load Balancer being provisioned** and how to configure **external access** so that all services are reachable (through NGINX) by the external world. 

The basic setup of the K8S cluster and Calico are taken from [this website](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-public-cloud/gce).

Note that this is **far from being perfect** and there are probably much better ways to achieve the same result with Calico alone or with bare metal load balancers like MetaLB, but I'm really not a networking expert, so this is my solution for now! 

The steps are going to be the following ones: 
* Step 1 : creation of the K8S cluster on GCE compute instances (including Calico), using `kubeadm` and creations of some basic services (REST API) <br/>
At this point you'll have a working cluster and the REST APIs will be reachable from within the cluster
* Step 2 : installing NGINX as an Ingress Controller and exposing it as NodePort and exposing the basic services (REST API) through NGINX Ingress <br/>
At this point the REST APIs will be reachable from the internet, using the external IP address of any kubernetes node (worker or controller)
* Step 3 : creating an NGINX Load Balancer external to the cluster to balance the traffic to the cluster nodes <br/>
At this point the REST APIs will be reachable from the internet, using the external IP address of the compute instance where the NGINX Load Balancer is installed

## Step 1 : K8S Cluster Creation
### 1.1. Create the VPC network
Everthing starts with a VPC network and a subnet. Check GCP documentation to understand the basics around VPC and subnets on GCP. 
 
```
gcloud compute networks create kubenet --subnet-mode custom

gcloud compute networks subnets create ksub \
  --network kubenet \
  --range 10.240.0.0/24
```

Now let's create the firewall rules that:
* will allow all internal traffic between kubernetes nodes
* will allow specific incoming traffic: access to the Kubernetes API Server

```
gcloud compute firewall-rules create kube-allow-internal \
  --allow tcp,udp,icmp,ipip \
  --network kubenet \
  --source-ranges 10.240.0.0/24

gcloud compute firewall-rules create kube-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubenet \
  --source-ranges 0.0.0.0/0
```
Cool, your network is created! <br>
Let's move to the next step! 

### 1.2. Create the Compute Instances
We're going to create the compute instances needed for the K8S cluster. Note that I'll only create one controller (for now).
```
gcloud compute instances create controller-0 \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-2 \
    --private-network-ip 10.240.0.11 \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet ksub \
    --zone europe-west1-c \
    --tags kube,controller

for i in 0 1; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 50GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet ksub \
    --zone europe-west1-c \
    --tags kube,worker
done
```

## 1.3. Install Docker and the kube tools
Kubernetes needs a CRI. <br>
On each compute instance (controller and worker nodes), we'll install Docker (mostly because it's the easiest to install, but it will work the same with containerd, there are just a few more steps).

On each node (controller and worker nodes), perform the following:
``` 
{
sudo apt update
sudo apt install -y docker.io 
sudo systemctl enable docker.service
sudo apt install -y apt-transport-https curl
}
```

Now let's install the Kubernetes base tooling.

Install kubeadm, kubelet, and kubectl on each node (controller and worker nodes):
```
{
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
}
```

### 1.4. Initialize the cluster
Now let's initialize the cluster. <br>
That's really easy: run this **only** on the **controller**: 
```
sudo kubeadm init --pod-network-cidr 192.168.0.0/16
```

Then create the kube config, that `kubectl` will need to connect to the API server: 
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now you what each **worker node** to join the cluster. <br>
That's where kubeadm join kicks in. Run this on each **worker node**:
```
sudo kubeadm join 10.240.0.11:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### 1.5. Install Calico
Finally, install **Calico**, to manage the networking between your nodes: 
```
curl https://docs.projectcalico.org/manifests/calico.yaml -O

kubectl apply -f calico.yaml
```

### 1.5. Create a simple REST API a test it
Now we're going to create a really simple REST API. For this, just `kubectl apply` the following: 
```
kubectl apply -f 

Other links: 
 * [Everything you need to know about K8S on Google Cloud](https://www.projectcalico.org/everything-you-need-to-know-about-kubernetes-networking-on-google-cloud/)
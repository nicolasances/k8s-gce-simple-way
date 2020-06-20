This is a walkthough on how to create a K8S cluster on Google Compute Engine instances (not using GKE), how to **configure NGINX as an Ingress Controller without Load Balancer being provisioned** and how to configure **external access** so that all services are reachable (through NGINX) by the external world. 

The basic setup of the K8S cluster and Calico are taken from [this website](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-public-cloud/gce).

Note that this is **far from being perfect** and there are probably much better ways to achieve the same result with Calico alone or with bare metal load balancers like MetaLB, but I'm really not a networking expert, so this is my solution for now! 

The steps are going to be the following ones: 
* **[Step 1](01.md)** : **creation of the K8S cluster** on GCE compute instances (including Calico), using `kubeadm` and creations of some basic services (REST API) <br/>
*At this point you'll have a working cluster and the REST APIs will be reachable from within the cluster.*
* **[Step 2](02.md)** : **installing NGINX as an Ingress Controller** and exposing it as NodePort and exposing the basic services (REST API) through NGINX Ingress <br/>
*At this point the REST APIs will be reachable from the internet, using the external IP address of any kubernetes node (worker or controller).*
* **[Step 3](03.md)**: **creating an NGINX Load Balancer external to the cluster** to balance the traffic to the cluster nodes <br/>
*At this point the REST APIs will be reachable from the internet, using the external IP address of the compute instance where the NGINX Load Balancer is installed.*

Alright! [Let's start with Step 1](01.md)!)


Other links: 
 * [Everything you need to know about K8S on Google Cloud](https://www.projectcalico.org/everything-you-need-to-know-about-kubernetes-networking-on-google-cloud/)
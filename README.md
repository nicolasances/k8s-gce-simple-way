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


Useful resources that helped me: 
 * K8S : [Adding a name to the Kubernetes API Server Certificate](https://blog.scottlowe.org/2019/07/30/adding-a-name-to-kubernetes-api-server-certificate/)
 * K8S : [Self-managed K8S on GCE](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-public-cloud/gce)
 * K8S & NGINX : [Using NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/)
 * NGINX : [Configuring NGINX as a Load Balancer](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
 * NGINX : [Controlling NGINX](https://docs.nginx.com/nginx/admin-guide/basic-functionality/runtime-control/)
 * NGINX : [Installing NGINX on CentOS](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-centos-7)
 * NGINX : [Soving 502 Bad Gateway](https://stackoverflow.com/questions/23948527/13-permission-denied-while-connecting-to-upstreamnginx)
 * GCP : [Everything you need to know about K8S on Google Cloud](https://www.projectcalico.org/everything-you-need-to-know-about-kubernetes-networking-on-google-cloud/)
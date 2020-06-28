## Running MongoDB on the Cluster

This guide will help you create a compute disk, attach it to a worker nodes and use it as a Persistent Volume for MongoDB.

First of all, follow the [tutorial on setting up compute disk, attaching it to a worker node and mounting it to a pod]. You can name the disk as you wish and attach it to any worker node. <br>
Follow the tutorial until the creation of the PVC: create the PVC, but don't create the deployment, since we'll use here another image.<br>
We'll assume here that the disk **is attached to** `worker-0`. 

When following that tutorial: 
* Name the Persistent Volume `kdisk-mongo-vol` and set the `nodeAffinity` to `worker-0` (if that's what you chose)
* Name the PVC `kdisk-mongo-pvc`

## Deploy MongoDB and mount that volume to it
Now we assume that if you followed the tutorial, you have: 
* A Storage Class `local-storage` to be used for PVs to be attached to these kind of local disks
* A Persistent Volume called `kdisk-mongo-vol` that will explicitly have a `nodeAffinity` to `worker-0` (if that's the node you chose), since that's the node that has the additional disk
* A Persistent Volume Claim `kdisk-mongo-pvc` that will claim that volume. 

Now you will create the following: 
* A Deployment for MongoDB `mongo-dep` that will mount a volume using `kdisk-mongo-pvc`
* A Service `mongo` that will expose the MongoDB deployment
* A Deployment `kapi-mongo-api-dep` to provide a REST API to access MongoDB
* A Service `kapi-mongo-api-svc` that will expose that deployment

```
kubectl apply -f https://raw.githubusercontent.com/nicolasances/k8s-gce-simple-way/master/resources/mongo.yaml
kubectl apply -f https://raw.githubusercontent.com/nicolasances/k8s-gce-simple-way/master/resources/kapi-mongo-api.yaml
```

### Testing the deployed service
You can now try the service:
```
kubectl run -it alpine-1 --image=alpine -- sh
apk add curl
curl -X POST http://kapi-mongo-api-svc:8080/expenses -H 'Content-Type: application/json' --data '{"amount": 12.2, "description": "Groceries shopping"}'
```
You should get that response: 
```
{"id":<id of the expense>}
```

If you try: 
```
curl -X GET http://kapi-mongo-api-svc:8080/expenses
```
You should receive the list of expenses (in this case it will only contain the newly created object)

You can exit the pod now and delete it: 
```
exit
kubectl delete pos alpine-1
```

You're good! You have a working (but **simple**) MongoDB installed on your GCE-hosted K8S cluster! 
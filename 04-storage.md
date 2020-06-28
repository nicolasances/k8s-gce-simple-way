## Disks and Volumes

This guide will help you create compute disks, attach them to your worker nodes and use them as a Persistent Volume.

### Creating the Disk
In this section, we're going to create a disk on GCP and attach it to one of the K8S worker nodes.

Let's create a GCP Disk named `kdisk-1`: 
```
gcloud compute disks create kdisk-1 \
    --size=10GB \
    --type=pd-standard \
    --zone=europe-west1-c
```

Once created, we should attach the disk to a compute instance. Let's attach this disk to `worker-1`. 
```
gcloud compute instances attach-disk worker-1 \
    --disk kdisk-1
```

### Formatting the Disk
Newly created disks need to be formatted. We just attached the disk to `worker-1`, so now let's ssh into the node (`gcloud compute ssh worker-1`) and look at the available disks using `lsblk`. 

Running `lsblk` on `worker-1` should now show something like that: 
```
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0     7:0    0 115.2M  1 loop /snap/google-cloud-sdk/135
loop1     7:1    0    97M  1 loop /snap/core/9289
loop2     7:2    0    55M  1 loop /snap/core18/1754
loop3     7:3    0 108.7M  1 loop /snap/google-cloud-sdk/137
sda       8:0    0    50G  0 disk 
├─sda1    8:1    0  49.9G  0 part /
├─sda14   8:14   0     4M  0 part 
└─sda15   8:15   0   106M  0 part /boot/efi
sdb       8:16   0    10G  0 disk 
```

You can see that the `sdb` disk is the newly created one. 

Let's **format the disk now**. This is done using `mkfs`, as specified on [GCP guide to formatting disks](https://cloud.google.com/compute/docs/disks/add-persistent-disk#formatting):

```
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
```

Now create a directory that serves as the mount point for the new disk (in this case I've chosen the name `kdisk`) and then use the mount tool to mount the disk to the instance, and enable the discard option. <br>
Then, configure read and write permissions on the device. For this example, we'll grant write access to the device for all users.
```
sudo mkdir -p /mnt/disks/kdisk
sudo mount -o discard,defaults /dev/sdb /mnt/disks/kdisk
sudo chmod a+w /mnt/disks/kdisk
```

Finally and **important**: let's add the zonal persistent disk to the /etc/fstab file so that the device automatically mounts again when the instance restarts.
```
sudo cp /etc/fstab /etc/fstab.backup
sudo blkid /dev/sdb
```
This should output something like that: 
```
/dev/sdb: UUID="c3b5f106-e170-47b7-bf94-2a8a2107091c" TYPE="ext4"
```
Open the `/etc/fstab` file in a text editor and create an entry that includes the UUID. The file, when you open it, should contain something like 
```
LABEL=cloudimg-rootfs   /        ext4   defaults        0 0
LABEL=UEFI      /boot/efi       vfat    defaults        0 0
```
What you want to do, is add 
```
UUID=c3b5f106-e170-47b7-bf94-2a8a2107091c       /mnt/disks/kdisk        ext4    discard,defaults,NOFAIL_OPTION  0 2
```

so the file should then look like something like this: 
```
LABEL=cloudimg-rootfs   /        ext4   defaults        0 0
LABEL=UEFI      /boot/efi       vfat    defaults        0 0
UUID=c3b5f106-e170-47b7-bf94-2a8a2107091c       /mnt/disks/kdisk        ext4    discard,defaults,NOFAIL_OPTION  0 2
```

**Important!** If you detach this zonal persistent disk or create a snapshot from the boot disk for this instance, edit the `/etc/fstab` file and remove the entry for this zonal persistent disk. Even with the `NOFAIL_OPTION` set to nofail or nobootwait, keep the `/etc/fstab` file in sync with the devices that are attached to your instance and remove these entries before you create your boot disk snapshot or when you detach zonal persistent disks.

## Deploying a Pod and mounting that volume to it
Now for a more realistic test: let's deploy a Pod and mount a volume that uses that disk.

First of all, let's clarify what we're going to use. We're going to use [K8S Local Persistent Volumes](https://kubernetes.io/blog/2019/04/04/kubernetes-1.14-local-persistent-volumes-ga) for this (we're not using HostPath volumes, as it's considered bad practice on K8S, as shown in the article attached).

Another **great advantage** of Local Persistent Volumes (that make them different from HostPath volumes) is that K8S will schedule your pods on nodes with the mounted disk that is referred by the PV, so that you're sure that you won't end up in a situation where the pod get scheduled on a node that doesn't have the mounted disk you're expecting. 

We're going to use a manifest that I created, that will deploy:
* A Storage Class `local-storage` to be used for PVs to be attached to these kind of local disks
* A Persistent Volume called `kdisk-vol` that will explicitly have a `nodeAffinity` to `worker-1`, since that's the node that has the additional disk
* A Persistent Volume Claim `kdisk-pvc` that will claim that volume. 
* A Deployment `kapi-vol-dep` that will mount a volume using `kdisk-pvc`
* A Service `kapi-vol-svc` that will expose that deployment

```
kubectl apply -f https://raw.githubusercontent.com/nicolasances/k8s-gce-simple-way/master/resources/kapi-vol.yaml
```

##### Some important notes to understand what's happening.
When defining the `Persistent Volume`, the `nodeAffinity` property is really important. In the example, I'm making sure that that Persistent Volume refers to `worker-1` node, since that's where we mounted the external disk.
```
  nodeAffinity:
      required: 
        nodeSelectorTerms: 
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In 
              values: 
              - worker-1
```
In the `Persistent Volume Claim`, you want to make sure to specify the `storageClassName` to make sure that it picks a PV that is bound to a local storage (`Local Persistent Volume`).
```
  storageClassName: local-storage
```
The rest is pretty standard! 

### Testing the deployed service
You can now try the service:
```
kubectl run -it alpine-1 --image=alpine -- sh
apk add curl
curl -X POST http://kapi-vol:8080/data
```
You should get that response: 
```
{"written":true}
```

You can exit the pod now and delete it: 
```
exit
kubectl delete pos alpine-1
```
If you `ssh` into `worker-1`, you'll see that under `/mnt/disks/kdisk` there's now a file `helloworld.txt`. It worked!

What's next? <br>
Why not [installing MongoDB on a cluster using Local Persistent Volumes](04-mongo.md)?

---
References: 
 * GCP : [Adding persistent disks to a running compute instance](https://cloud.google.com/compute/docs/disks/add-persistent-disk)

Here is short HOWTO to maintain and troubleshoot GKE and cryptonodes.

You may also use [official doc](https://cloud.google.com/kubernetes-engine/docs/troubleshooting) to troubleshoot GKE cluster.

We assume kubectl default context is configured correctly to connect to the required cluster. 

### Diagnose and troubleshooting

Locate required pods
```bash
kubectl get pod
```
Check pod logs (`-f` to follow)
```bash
kubectl logs -f tezos-0-0 --tail=10
```
Check pod info to troubleshoot startup problems, liveness check problems, etc:
```bash
kubectl describe pod tezos-0-0
```
Restart pod if hung
```bash
kubectl delete pod tezos-0-0
```
Check allocated disk size 
```bash
kubectl get pvc
``` 
Shell exec to container to check/troubleshoot from inside
```bash
kubectl exec -it tezos-0-0 sh
``` 

### Upgrade cryptonode version 

Let's assume you need to upgrade Tezos from `v7-release` to `v8-release`. Here is what you need to do:
* update `values-dev.yaml` you used before to deploy Tezos or create new `values-dev.yaml` file with following content  
```yaml
image:
  repository: tezos/tezos
  tag: v8-release
```
* upgrade Tezos helm release in the cluster, we use release named `tezos-0` in example below
```bash
cd tezos-kubernetes
helm upgrade tezos-0 charts/tezos/ --reuse-values --force --atomic --values values-dev.yaml
```

### Snapshot disk with blockchain
Let's assume you need to snapshot disk from pod `tezos-0-0`. First we need to find what disk is actually used by this pod
```bash
kubectl describe pod tezos-0-0
```
Check output for Volumes, my case is
```yaml
Volumes:
  tezos-pvc:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  tezos-pvc-tezos-0-0
    ReadOnly:   false
```
We get `ClaimName: tezos-pvc-tezos-0-0`, now we need to find corresponding `PersistentVolume`:
```bash
kubectl describe pvc tezos-pvc-tezos-0-0
``` 
Check output for Volumes, my case is
```yaml
Volume:        pvc-b2148859-fb11-4513-afa1-fd9f5c8c4d82
```
And now we need to get disk name from PV
```bash
kubectl describe pv pvc-b2148859-fb11-4513-afa1-fd9f5c8c4d82
```
Check output for `Source`
```yaml
Source:
    Type:       GCEPersistentDisk (a Persistent Disk resource in Google Compute Engine)
    PDName:     gke-tezos-node-1-f9835-pvc-b2148859-fb11-4513-afa1-fd9f5c8c4d82
``` 
We get `gke-tezos-node-1-f9835-pvc-b2148859-fb11-4513-afa1-fd9f5c8c4d82`, that's the name of disk we need to snapshot.
You may use [official doc](https://cloud.google.com/compute/docs/disks/create-snapshots) to create snapshot, here is quick example command to do so
```bash
gcloud compute disks snapshot gke-tezos-node-1-f9835-pvc-b2148859-fb11-4513-afa1-fd9f5c8c4d82 --snapshot-names=tezos-0-snapshot
```
It may be better to stop blockchain node [to get consistent snapshot with high probability](https://cloud.google.com/compute/docs/disks/snapshot-best-practices). Here is how you can do it for example with Tezos node:
```bash
kubectl scale statefulset tezos-0 --replicas=0
``` 
Wait 1 minute and then create the snapshot. Use following command to start node again:
```bash
kubectl scale statefulset tezos-0 --replicas=1
```
You may need to convert your snapshot to an image, for example to share the image
```bash
gcloud compute images create tezos-2020-06-01 --source-snapshot=tezos-0-snapshot
``` 
### Provision cryptonode with pre-existing image
When you have someone who shared pre-synced cryptonode disk image with you, you can create a new disk from this image and use it with your cryptonode, and here is how.
* (optional) copy disk image to your project
```bash
gcloud compute images create tezos-2020-06-01 --source-image=tezos-2020-06-01 --source-image-project=<SOURCE-PROJECT>
```

* just create SSD disk from the image, pay attention to zone, it must be the same as your GKE cluster 
```bash
gcloud compute disks create tezos-0 --type pd-ssd --zone us-central1-a  --image=tezos-2020-06-01 --image-project=<SOURCE-PROJECT>
```

#### Attaching disk to cryptonode pod
Now we need to create [PersistentVolume in Kubernetes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)(PV) and use this PV with [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) (PVC) we already have.
* adjust [pv.yaml](pv.yaml) with your disk name, zones etc. In this manual we assume you have required storage classes already from cryptonode deployment.
* create PV via following command, use one of them:
```bash
kubectl create -f pv.yaml
```
* shutdown cryptonode:
```bash
kubectl scale statefulset tezos-0 --replicas=0
``` 
give it some time to shutdown, you can monitor it with `kubectl get pod -w` usually

* replace existing PVC by a copy with another disk name `tezos-0`, I use `tezos-pvc-0` PVC in the example below:
```bash
# backup just in case
kubectl get pvc tezos-pvc-0 -o yaml > tezos-pvc-0.yaml 
kubectl get pvc tezos-pvc-0 -o json|jq '.spec.volumeName="tezos-0"'| kubectl replace --force -f -
```
* start cryptonode up and check logs
```bash
kubectl scale statefulset tezos-0 --replicas=1
kubectl get pod -w
kubectl logs -f tezos-0-0
``` 

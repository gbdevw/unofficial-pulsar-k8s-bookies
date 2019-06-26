## Unofficial alternative deployment for Apache Pulsar Bookies on Kubernetes
Alternative method to deploy bookies for Apache Pulsar on Kubernetes using Local Persistent Volumes and Stateful Sets.

### Problem & Motivation

The current provided way to deploy Apache Pulsar Bookies uses a Demon Set to deploy one bookie on each node of your K8S cluster and HostPath volumes to provide local storage needed by the bookies. This method poses several potential problems :

1. You cannot choose which nodes are elligible for bookie scheduling
2. You cannot choose the number of bookies you deploy nor the number of bookie each node will host
3. HostPath requires that you can use a directory/partition/disk the same way one each node. This is not always true
4. HostPath do not use K8S mechanisms to manage storage (Persistent Volumes & Persistent Volume Claims) 

Since Kubernetes 1.14 (March 2019), it is possible to use Local Persistent Volumes to provide local storage through the Persistent Volume and Persistent Volume Claims mechanisms. The binding of Local Persistent Volumes is topology-aware, meaning that K8S will bind local storage attached to the node the pod is scheduled on.

The alternative method leverage this new functionality to ease the tunning of bookie deployment and scaling.

### How it works

Here are the abstract steps to use the proposed method :

1. Create the Storage Class that will be attached to Local Persistent Volumes
2. Provision Local Persistent Volumes on nodes that will host bookies, you need two volumes per bookie
3. Label the nodes that can provide local storage and host bookies
4. Deploy the Stateful Set that will schedule one bookie on one elligible node
5. Scale the Stateful Set. The Stateful set will schedule new bookies

The scheduling rules are :

1. Bookies are scheduled on nodes that are labelled and signal that they can provide local storage / host bookies
2. Only one bookie is allowed per node (this can be changed to allow multiple bookies on the same node)

Once the bookie is scheduled, a local storage attached to the node the bookie is scheduled on is bound. The Stateful Set provides a sticky identity to the bookie. If the bookie needs to be rescheduled, it will be rescheduled on the same node with the same storage.

### Usage

1. Create the Storage Class that will be attached to Local Persistent Volumes

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-k8s-bookies/master/fast-local-storage-class.yaml
```

2. Provision Local Persistent Volumes on nodes that will host bookies, you need two volumes per bookie

Here is an example that creates two 10 Gi local persistent volumes on one node using the storage class created. The storage will write on two logical partitions (or disks) mounted on /bookie1 & /bookie2 :

```yaml
# First local persistent volume that should be dedicated to a bookie hosted on the node
apiVersion: v1
kind: PersistentVolume

metadata:
  name: worker1-local-bookie1
 
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-local
  local:
    path: /bookie1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ubuntu-k8s-worker-1
---

# Second local persistent volume that should be dedicated to a bookie hosted on the node
apiVersion: v1
kind: PersistentVolume

metadata:
  name: worker1-local-bookie2
  
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-local
  local:
    path: /bookie2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ubuntu-k8s-worker-1
```

3. Label the nodes that can provide local storage and host bookies

In the script, a custom label 'has-local-fast-storage' is used with the value 'yes'

```bash
kubectl label nodes <node_name> has-local-fast-storage=yes
```

4. Deploy the Stateful Set that will schedule one bookie on one elligible node

```bash
kubectl apply -f https://raw.githubusercontent.com/guillaume-braibant/unofficial-pulsar-k8s-bookies/master/bookie-alternative.yaml
```

5. Scale the Stateful Set

```bash
kubectl scale statefulset bookie --replicas=2
```

### Known issues

1. You must NOT ask for more bookies than your cluster can handle when deploying the stateful set

For example, if your cluster can handle two bookies and that you ask for 3 replicas during the INITIAL deployment of the stateful set, you will run into troubles : Bookies will be stuck in a crash loop (reason unknown).

However, if you scale the stateful set and ask for more bookies than the cluster can handle, bookies will continue to be OK. Unnecessary bookies will remain in pending state until some node become elligible for their scheduling.

2. There is not automatic clean up of disks once the claim for a local persistent volume is deleted

To reclaim storage space on your disks, you first need to delete the persistent volume claims that are bound to your local persistent volumes. Then, you have to manually clean your disks.

A project is currently developed in order to allow automatic provisionning of local persistent volumes and automatic disk clean up once the persistent volume claim bound to the local persistent volume is deleted (you still have to manually delete the pvc).

https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner

### Pitfalls

1. Be sure to require/allocate enough storage for the bookie journal

Default bookie journal configuration requires to keep 5 files (journalMaxSizeMB) of 2GB (journalMaxSizeMB). Thus, storage dedicated to bookie journal should provide more than 10GB if you use default configuration. When a bookie does not have enough space to keep all journals, it crashes.

2. Be sure to require and allocate enough storage for the bookie ledgers

Default bookie journal configuration require a minimal storage space available (1 or 2GB if I remeber) in order to create a ledger. Once a bookie run out of storage and cannot create new ledgers, it switches in read-only mode. If all your bookies are in read-only modes, you will not be able to send more messages through Pulsar. Be sure to allocate/require enough storage to handle your load.

### Tuning

This script relies on affinity rules to tune the scheduling of bookies (strict 1 bookie per bookie node). There are a lot of affinity rules you can prefer to those that are used. In the next Kubernetes versions, more affinity rules will be added. They might fit better your use case.

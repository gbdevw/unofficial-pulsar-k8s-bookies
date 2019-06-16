## Unofficial alternative deployement for Apache Pulsar Bookies on Kubernetes
Alternative method to deploy bookies for Apache Pulsar on Kubernetes using Local Persistent Volumes and Stateful Sets.

### Problem & Motivation

The current provided way to deploy Apache Pulsar Bookies uses a Demon Set to deploy one bookie on each node of your K8S cluster and HostPath volumes to provide local storage needed by the bookies. This method poses several potential problems :

1. You cannot choose which nodes are eligible for bookie scheduling
2. You cannot choose the number of bookies you deploy nor the number of bookie each node will host
3. HostPath requires that you can use a directory/partition/disk the same way one each node. This is not always true
4. HostPath do not use K8S mechanisms to manage storage (Persistent Volumes & Persistent Volume Claims) 

Since Kubernetes 1.14 (March 2019), it is possible to use Local Persistent Volumes to provide local storage through the Persistent Volume and Persistent Volume Claims mechanisms. The binding of Local Persistent Volumes is topology-aware, meaning that K8S will bind local storage attached to the node the pod is scheduled on.

The alternative method proposed leverage this new functionality in order to ease the tunning of bookie deployment and scaling.

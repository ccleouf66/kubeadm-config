# Rook-ceph deployment

## Configuration

The operator is in charge to deploy ceph CSI plugin.
For this configuration, we enable `readAffinity` feature. This instuct the RBD kernel module (KRBD) to allow serving reads from an OSD closest to the client, according to OSD locations defined in the CRUSH map and topology labels on nodes.

In this case the cluster is stretch across 3 differents regions with a latency that can up to ~ 10-12ms.
If the read operations are made one the local node that can drastically increase the performance and latency.

That mean we will have one replicas of the data on each node and so the guarantee to have one replicas locally.

If additional node join the ceph cluster, we can adjust the logic and granularity with more label.
The label on the kubernetes node will be used when new OSD join the cluster and added to the ceph crush map with the infos.

### label the node 
```bash
kubectl label node scale3-rbx topology.kubernetes.io/region=eu
kubectl label node scale3-rbx topology.kubernetes.io/zone=eu-rbx

kubectl label node scale3-sbg topology.kubernetes.io/region=eu
kubectl label node scale3-sbg topology.kubernetes.io/zone=eu-sbg

kubectl label node scale3-lim topology.kubernetes.io/region=eu
kubectl label node scale3-lim topology.kubernetes.io/zone=eu-lim
```

When setting the node labels prior to CephCluster creation, these settings take immediate effect. However, applying this to an already deployed CephCluster requires removing each node from the cluster first and then re-adding it with new configuration to take effect. Do this node by node to keep your data safe! Check the result with ceph osd tree from the Rook Toolbox. The OSD tree should display the hierarchy for the nodes that already have been re-added.

This labels are used for:
- replication strategy in cluster values
- readAffinity strategy in operator values for CSI plugin configuration

replication:
```yaml
cephBlockPools:
- name: ceph-blockpool-replicated
  spec:
    failureDomain: host # host field will be automatically added in the crushmap by ceph
    replicated:
      size: 3
```

readAffinity:

```yaml
...
  topology:
    # -- Enable topology based provisioning
    enabled: true
    # domainLabels define which node labels to use as domains
    # for CSI nodeplugins to advertise their domains
    domainLabels:
    - kubernetes.io/hostname
    - topology.kubernetes.io/zone
    - topology.kubernetes.io/region

  readAffinity:
    enabled: true
    # -- Define which node labels to use
    # as CRUSH location. This should correspond to the values set
    # in the CRUSH map.
    # @default -- labels listed [here](../CRDs/Cluster/ceph-cluster-crd.md#osd-topology)
    crushLocationLabels:
    - topology.kubernetes.io/region
    - topology.kubernetes.io/zone
...
```

## Install the rook-ceph operator
```bash
helm repo add rook-release https://charts.rook.io/release
helm install -n rook-ceph --create-namespace -f rook-ceph-values.yaml rook-ceph  rook-release/rook-ceph
```

## Deploy Ceph cluster
```bash
helm install -n rook-ceph --create-namespace -f values-ceph-cluster.yaml rook-ceph-cluster rook-release/rook-ceph-cluster
```

## Result

### Read test without readAffinity configuration

~ 200MiB/s.

### Read test with readAffinity configuration:

READ: bw=2455MiB/s (2574MB/s), 2455MiB/s-2455MiB/s (2574MB/s-2574MB/s), io=288GiB (309GB), run=120053-120053msec -> SBG
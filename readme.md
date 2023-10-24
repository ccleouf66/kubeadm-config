# Kubernetes cluster deployment

This repository host configurations to deploy a Kubernetes cluster cluster. This cover the configuration of the full stack in order to provide best performances and a good evoluvity.

The configurations are applied on OVHcloud but can be deployed on any other enviroment. The services used for the samples are :
- Public cloud for
    - Control Plane and ETCD nodes
    - Control Plane Loadbalancer
- Bare Metal server for nodes
- vRack for communication across different region and OVHcloud universes

The Kubernetes cluster is deployed across multiple region to provide HA and fault tolerance.

The solution used are :
- [Kubeadm to deploye a v1.28.0 Kubernetes cluster](./kubeadm.md)
- OS ubuntu Ubuntu 22.04.2 LTS - 5.15.0-87-generic
- [CRI: containerd://1.6.24](./kubeadm.md)
- [CNI Calico in BGP mode](./calico.md)
- Rook Ceph for the storage solution
- [Network tuning at kernel level to exploit 25gbps](./sd-cross-regions-vrack.md)

# Kubernetes cluster upgrade

Upgrade [v1.27.x -> v1.28.x](./upgrade.md)

# Calico configuration

## Objective
The objective is to use Calico as Kubernetes network plugin. To provide best performance, following Calico recommendation, BGP mode is choosen without overlay.

## Environment
Kubernetes v1.25.0
Calico 3.24.1
OS ubuntu Ubuntu 22.04.1 LTS
vRack *pcc145,146,3ds* in CA environment.  
The dedicated servers are integrated in the vRack.
The Public Cloud project is integrated in the vRack.
The private network used is *cyril-net*: *VLAN 1907* - *10.119.0.0/16*.

## Public Cloud VM servers
3 VMs are provisioned on a Public Cloud project in 3 different regions.  
Each server use *SCALE-3*:

Each VM use *General Purpose B2-7* flavor :

| Component    | Configuration |
| ----------- | ----------- |
| CPU | 2 vCores (2.3GHz) |
| RAM | 7GB |
| System Disk | 1x SSD SATA 50GB |
| Public Network| 250 Mbit/s unmetered and guaranteed |
| Private Network | 300 Gbit/s unmetered |

And the 3 servers characteristics:

| Hostname    | Region | Public IP | Private IP |
| ----------- | ----------- | -- | -- |
| m1 | DE1  | xxx.xxx.xxx.xxx | 10.119.25.18 |
| m2 | GRA9 | xxx.xxx.xxx.xxx | 10.119.95.181 |
| m3 | SBG5 | xxx.xxx.xxx.xxx | 10.119.117.234 |

## Dedicated servers
3 dedicated servers are provisioned, in 3 different regions.  
Each server is *SCALE-3*:

| Component    | Configuration |
| ----------- | ----------- |
| CPU | AMD Epyc 7642 - 48c/96t - 2.3GHz/3.3GHz |
| RAM | 256GB DDR4 ECC 2933MHz |
| System Disk | 2x SSD SATA 480GB Soft RAID |
| Data Disk | 2×1.92 TB SSD NVMe |
| Public Network| 1Gbit/s unmetered and guaranteed |
| Private Network | 25Gbit/s unmetered and guaranteed |

And the 3 servers characteristics:

| Hostname    | Region | Public IP | Private IP |
| ----------- | ----------- | -- | -- |
| lejav-scale3-lim | LIM | xxx.xxx.xxx.xxx| 10.119.250.101 |
| lejav-scale3-sbg | SBG |xxx.xxx.xxx.xxx | 10.119.250.102 |
| lejav-scale3-rbx | RBX | xxx.xxx.xxx.xxx| 10.119.250.103 |

## Deployement
Calico is deployed with de Operator:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/tigera-operator.yaml
```

The following configuration is applied:
```yaml
---
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 25
      cidr: 192.168.0.0/16
      encapsulation: None
      natOutgoing: Enabled
      nodeSelector: all()
    # linuxDataplane: BPF
    mtu: 9000
    bgp: Enabled
    nodeAddressAutodetectionV4:
      kubernetes: NodeInternalIP

---
# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer 
metadata: 
  name: default 
spec: {}
```

### Check BGP config
On each node install calicoctl cli:
```bash
curl -L https://github.com/projectcalico/calico/releases/download/v3.24.1/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
sudo mv ./calicoctl /usr/local/bin/
```

Then check the BGP status:
```bash
sudo calicoctl node status
```

The connexion STATE should be *up*:
```bash
Calico process is running.

IPv4 BGP status
+----------------+-------------------+-------+----------+-------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+----------------+-------------------+-------+----------+-------------+
| 10.119.25.18   | node-to-node mesh | up    | 18:59:41 | Established |
| 10.119.95.181  | node-to-node mesh | up    | 18:59:45 | Established |
| 10.119.250.101 | node-to-node mesh | up    | 18:59:43 | Established |
| 10.119.250.103 | node-to-node mesh | up    | 18:59:42 | Established |
| 10.119.117.234 | node-to-node mesh | up    | 18:59:45 | Established |
+----------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

### Security policies

Calico GlobalNetworkPolicy to isolate a namespace : [template](./tools/gnp-isolate-ns.yaml).
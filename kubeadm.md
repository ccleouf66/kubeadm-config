# Create a Kubernetes cluster with kubeadm

- Kubernetes version: v1.32.0
- OS: ubuntu Ubuntu 24.04.1 LTS
- CRI: containerd://2.0.1

### Update OS

```bash
sudo apt update -y && sudo apt upgrade -y
```

### Install containerd

```bash
curl -O -JL https://github.com/containerd/containerd/releases/download/v2.0.1/containerd-2.0.1-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-2.0.1-linux-amd64.tar.gz
sudo mkdir -p /usr/local/lib/systemd/system/
sudo curl -o /usr/local/lib/systemd/system/containerd.service -JL https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### Install runc
```bash
sudo mkdir -p /usr/local/sbin
sudo curl -o /usr/local/sbin/runc -JL https://github.com/opencontainers/runc/releases/download/v1.2.4/runc.amd64
sudo chmod a+rx /usr/local/sbin/runc
```

### Configure containerd to use systemd cgroup
```
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
```

Add the following line: `SystemdCgroup = true`

```yaml
...
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
  SystemdCgroup = true
...
```
```bash
sudo systemctl restart containerd
```

Note: kubelet has to be configured as well later on

### Install crictl
```bash
VERSION="v1.32.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
sudo tar Czxvf /usr/local/bin crictl-$VERSION-linux-amd64.tar.gz
```

### OS config

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Disable swap
sudo swapoff -a
sudo sed -i '/[^V^I ] swap[^V^I ]/ s/^/#/' /etc/fstab
sudo grep swap /etc/fstab
```
Check that all lines for swap are correctly commented.

Disable IPv6 (optional)  
It may help to completely disable IPv6. For instance, for Ubuntu 22.04, follow this guide: https://linuxconfig.org/how-to-disable-ipv6-address-on-ubuntu-22-04-lts-jammy-jellyfish.


### Install kubeadm kubectl and kubelet
```bash

sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```


### Configure octavia LB

```bash
openstack loadbalancer create --name <kube-lb> --flavor small --vip-subnet-id <private_subnet_id>
openstack floating ip create Ext-Net
openstack floating ip set --port <lb_vip_port_id> <floating_ip>

openstack loadbalancer listener create --name <kube-apiserver> --protocol TCP --protocol-port 6443 <kube-lb>
openstack loadbalancer pool create --name <master-nodes> --lb-algorithm ROUND_ROBIN --listener <kube-apiserver> --protocol TCP
openstack loadbalancer healthmonitor create --delay 5 --max-retries 4 --timeout 10 --type TCP <master-nodes>

openstack loadbalancer member create --subnet-id <private_subnet_id> --address <master_1_private_ip> --protocol-port 6443 <master-nodes>
openstack loadbalancer member create --subnet-id <private_subnet_id> --address <master_2_private_ip> --protocol-port 6443 <master-nodes>
openstack loadbalancer member create --subnet-id <private_subnet_id> --address <master_3_private_ip> --protocol-port 6443 <master-nodes>
```

### Create Kubeadm-config.yaml
```yaml
---
apiVersion: kubeadm.k8s.io/v1beta4
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef            # Change the token
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: <master_1_private_ip>
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: master-1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane

---
apiServer:
  timeoutForControlPlane: 4m0s
  certSANs:       # SANS to add to the kube-apiserver SSL certs
  - X.X.X.X       # master_1_private_ip
  - X.X.X.X       # master_2_private_ip
  - X.X.X.X       # master_3_private_ip
  - X.X.X.X       # LB public ip
  - X.X.X.X       # LB private ip
  - lb-CNAME      # LB DNS record
apiVersion: kubeadm.k8s.io/v1beta4
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: 1.32.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 192.168.0.0/16
scheduler: {}
controlPlaneEndpoint: X.X.X.X:6443   #Private IP of the LB

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
serverTLSBootstrap: true      # for metrics server
systemReserved:
  cpu: 500m
  memory: 512mi
kubeReserved:
  cpu: 500m
  memory: 2G
cpuManagerPolicy: static
cpuManagerPolicyOptions:
  full-pcpus-only: "true"
```

### Init cluster on first nodes only
```bash
sudo kubeadm init --upload-certs --config kubeadm-config.yaml
```

### Approve CSRs
```bash
kubectl get csr
NAME        AGE    SIGNERNAME                                    REQUESTOR               REQUESTEDDURATION   CONDITION
csr-xxxxx   39m    kubernetes.io/kubelet-serving                 system:node:<master1>   <none>              Pending
csr-yyyyy   39m    kubernetes.io/kubelet-serving                 system:node:<master1>   <none>              Pending
...
```
Approve each pending csr:
```bash
kubectl certificate approve csr-xxxxx csr-yyyyy ...
```


### Join the other master nodes
```bash
# master 2
sudo kubeadm join <master_1_private_ip>:6443 --token <bootstrap_token> \
        --discovery-token-ca-cert-hash <sha256:ca> \
        --control-plane --certificate-key <cert_key> \
        --apiserver-advertise-address <master_2_private_ip>

# master 3
sudo kubeadm join <master_1_private_ip>:6443 --token <bootstrap_token> \
        --discovery-token-ca-cert-hash <sha256:ca> \
        --control-plane --certificate-key <cert_key> \
        --apiserver-advertise-address <master_2_private_ip>
```

sudo kubeadm join 192.0.0.1:6443 --token rjtt1p.vwny2zxwr94hex48 --discovery-token-ca-cert-hash sha256:12b04a890791dd71669536f39ace5c7a3273e3a80840f201611d7613f6a56177 --control-plane --apiserver-advertise-address 192.0.0.3 --certificate-key 5bd7fdb4e8d1eb59e43988abc0317a4e2ca5620f94273d73a828977efded8421

### Add worker nodes

If you add additional workers after 24h the existing bootstrap token is probably expired. To generate new one:

```bash
kubeadm token create --print-join-command

kubeadm join <IP_LB>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<SHA256>
```

### Configure Node INTERNAL IP

By default, kubelet advertise the interface IP with the default gateway set on it as INTERNAL IP. This INTERNAL IP is used by kube-apiserver when it needs to communicate with the node. To use the private network instead (for security purpose) we can edit kubelet config on each node:

```bash
sudo vi /var/lib/kubelet/kubeadm-flags.env
```

```bash
KUBELET_KUBEADM_ARGS="--node-ip=<PRIVATE IP NODE> --container-runtime=remote --..."
```

### Approve CSRs
```bash
kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                 REQUESTEDDURATION   CONDITION
csr-mq6c9   80s     kubernetes.io/kubelet-serving                 system:node:<master2>     <none>              Pending
csr-pdw4k   6m15s   kubernetes.io/kubelet-serving                 system:node:<master3>     <none>              Pending
...
```
Approve each pending csr:
```bash
kubectl certificate approve csr-xxxxx csr-yyyyy ...
```

### Install the Network plugin

Here, we used Calico as Network plugin solution. You can follow our config in [calico.md](./calico.md).

### Install metrics-server

The Kubernetes metrics-server can be installed usign the official [git repo](https://github.com/kubernetes-sigs/metrics-server)

# Create a Kubernetes cluster with kubeadm

- Kubernetes version: v1.25.0
- OS: ubuntu Ubuntu 22.04.1 LTS
- CRI: containerd://1.6.8

### Update OS

```bash
sudo apt update -y && sudo apt upgrade -y
```

### Install containerd

```bash
curl -O -JL https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.6.8-linux-amd64.tar.gz
sudo mkdir -p /usr/local/lib/systemd/system/
sudo curl -o /usr/local/lib/systemd/system/containerd.service -JL https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### Install runc
```bash
sudo mkdir -p /usr/local/sbin
sudo curl -o /usr/local/sbin/runc -JL https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64
sudo chmod a+rx /usr/local/sbin/runc
```

### Configure containerd to use systemd cgroup
```
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo vi /etc/containerd/config.toml
```

```yaml
...
SystemdCgroup = true
...
```
```bash
sudo systemctl restart containerd
```

Note: kubelet has to be configured as well later on

### Install crictl
```bash
VERSION="v1.25.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
sudo tar Czxvf /usr/local crictl-$VERSION-linux-amd64.tar.gz
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
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Install kubeadm kubectl and kubelet
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
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
apiVersion: kubeadm.k8s.io/v1beta3
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
  taints: null

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
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: 1.25.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
controlPlaneEndpoint: X.X.X.X:6443   #Private IP of the LB

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
serverTLSBootstrap: true      # for metrics server
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

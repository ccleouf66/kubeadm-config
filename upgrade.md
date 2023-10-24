# Upgrade a Kubernetes cluster with kubeadm

- Kubernetes version: v1.27.x -> v1.28.x
- OS: ubuntu Ubuntu 22.04.2 LTS - 5.15.0-87-generic
- CRI v1: containerd://1.6.8 -> containerd://1.6.24
- runc 1.1.4 -> 1.1.9

# Node prerequisite upgrade

## Drain node

These steps are to prepare the nodes and have to be executed on each node one by one to not impact production.

```bash
kubectl drain <node-to-drain> --ignore-daemonsets --delete-emptydir-data
```

## OS update

```bash
sudo apt update -y && sudo apt upgrade -y
```

## Upgrade containerd and runc

You can check rommended and compatibility versions for containerd and kubernetes here : https://containerd.io/releases/

Containerd v1.6 is the LTS version (oct. 2023).
For Kubernetes v1.28 Containerd v1.6.15+ is recommended or v1.7.0+

For production purpose we go with latest Containerd LTS version -> v1.6.24(oct. 2023).

cri-containerd-1.6.24 comme with:

- containerd v1.6.24
- runc v1.1.9
- crictl cli
- ctr cli

If not yet up to date: 

```bash
wget https://github.com/containerd/containerd/releases/download/v1.6.24/cri-containerd-1.6.24-linux-amd64.tar.gz
 
sudo tar --no-overwrite-dir -C / -xzf cri-containerd-1.6.24-linux-amd64.tar.gz
sudo systemctl restart containerd
```

Validation:
``` bash
sudo ctr version
```
```bash
Client:
  Version:  v1.6.24
  Revision: 61f9fd88f79f081d64d6fa3bb1a0dc71ec870523
  Go version: go1.20.8

Server:
  Version:  v1.6.24
  Revision: 61f9fd88f79f081d64d6fa3bb1a0dc71ec870523
  UUID: 01f18487-d47d-458f-a59e-b42b3ed6837c
```
```bash
sudo runc -v
```
```bash
runc version 1.1.9
commit: v1.1.9-0-gccaecfcb
spec: 1.0.2-dev
go: go1.20.8
libseccomp: 2.5.3
```

Update sandbox image:
```bash
sudo vi /etc/containerd/config.toml
```
```toml
#sandbox_image = k8s.gcr.io/pause:3.6
sandbox_image = "registry.k8s.io/pause:3.9"
```
```bash
sudo systemctl restart containerd
```

Finally you can reboot and uncordon the node to apply kernel upgrade.
```bash
kubectl uncordon <node-to-uncordon>
```

# Master upgrade

## Upgrade control plan parts

### Upgrade kubeadm (all node)
```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# update package information & search available kubeadm versions
sudo apt update
sudo apt-cache madison kubeadm

sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.28.3-1.1
sudo apt-mark hold kubeadm
kubeadm version
```

### Upgrade control plan components (first master only)

```bash
sudo kubeadm upgrade plan
```

Validate the target versions and proceed to the upgrade:
```bash
sudo kubeadm upgrade apply v1.28.3
```

### Upgrade control plan components (other master only)

```bash
sudo kubeadm upgrade node
```

# Upgrade kubelet on all nodes

You can start on 3 master nodes and finally upgrade worker nodes one by one with drain & uncordon proccess to ensure the services availability.

## Master nodes

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubelet=1.28.3-1.1 kubectl=1.28.3-1.1
sudo apt-mark hold kubelet kubectl
```
Restart the kubelet:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

At this step, the master nodes should be updated:
```bash
NAME               STATUS   ROLES           AGE    VERSION
lim-m1          Ready    control-plane   403d   v1.27.6
rbx-m2          Ready    control-plane   403d   v1.27.6
sbg-m3          Ready    control-plane   403d   v1.27.6
scale3-lim      Ready    <none>          390d   v1.26.0
scale3-rbx      Ready    <none>          357d   v1.26.0
scale3-sbg      Ready    <none>          357d   v1.26.0
```

## Worker nodes

### Drain node
```bash
kubectl drain <node-to-drain> --ignore-daemonsets --delete-emptydir-data
```

### Update kubelet & kubeadmkuebct  

```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-mark unhold kubeadm kubelet
sudo apt-get install -y kubeadm=1.28.3-1.1 kubelet=1.28.3-1.1
sudo apt-mark hold kubeadm kubelet
```

Restart the kubelet:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Uncordon the node.
```bash
kubectl uncordon <node-to-uncordon>
```

# cluster

My Kubernetes cluster bootstrap configuration

Tested on Ubuntu 18.04 (Bionic Beaver)

## Cluster setup

Save your node's local IP address to a variable before continuing.

```
export NODE_LOCAL_IP=<local ip>
```

Step by step...

```bash
apt update
apt-get install apt-transport-https ca-certificates curl software-properties-common curl

# Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Kubernetes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
echo 'deb https://apt.kubernetes.io/ kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl docker-ce=18.06.0~ce~3-0~ubuntu
apt-mark hold kubelet kubeadm kubectl docker-ce

# Prepare for CNI
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.d/99-flannel.conf
echo 'net.bridge.bridge-nf-call-iptables = 1' >> /etc/sysctl.d/99-flannel.conf

# Kubernetes
kubeadm config images pull
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=${NODE_LOCAL_IP}

# A must for single node setup
kubectl taint nodes --all node-role.kubernetes.io/master-

# A must for life
kubectl completion bash >> /etc/bash_completion.d/kubernetes

# Docker config
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

### [Flannel][] (CNI)

```bash
kubectl apply -f flannel/
```

## Cluster add-ons

### [metrics-server][]

Must update *metrics-server-deployment.yaml* with the master node hostname
and local IP before deploying.

**Warning:** An extra argument `--kubelet-insecure-tls` is supplied to make
this work. The underlying issue should be fixed.

```bash
$ kubectl create -f metrics-server/
```


## References
- https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
- https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/
- https://kubernetes.io/docs/setup/independent/troubleshooting-kubeadm/
- https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet
- https://github.com/kubernetes-incubator/metrics-server/issues/131#issuecomment-418613256

[metrics-server]: https://github.com/kubernetes-incubator/metrics-server
[Flannel]: https://github.com/coreos/flannel
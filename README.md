# DBSync
Sync postgresql to postgresql using debezium and kafka connect, HA Kubernetes cluster

## Deploy Kafka on K8s on-premise

![image](https://github.com/toanlcgift/DBSync/assets/12400049/83d68616-427d-412c-abc5-99ec4892be21)

### Environment

K8s Node               | OS                              | RAM      | Total Processor Cores      | local IP | DNS |
-----------------------|-------------------------------------------|-----------------------------|-------------------------|-------------------------|-------------------------|
master             | `Ubuntu 22.04 LTS` | 4GB | 2 | 192.168.123.123 | localk8s.com |
node1             | `Ubuntu 22.04 LTS` | 4GB | 2 | 192.168.123.124 |  localnodek8s.com |

### Install kubelet, kubeadm, kubectl

``` bash
sudo apt -y install curl apt-transport-https
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt -y install vim git curl wget kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo systemctl enable kubelet
```

### Install docker

``` bash
sudo apt update
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker-archive-keyring.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io docker-ce docker-ce-cli
sudo mkdir -p /etc/systemd/system/docker.service.d

sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl daemon-reload 
sudo systemctl restart docker
sudo systemctl enable docker
```

### Install Mirantis cri-dockerd

``` bash
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
echo $VER

wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
tar xvf cri-dockerd-${VER}.amd64.tgz

sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
```

### Config & run kubernetes

``` bash
sudo kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock

#Master only Start
sudo kubeadm init --pod-network-cidr=172.24.0.0/16 --cri-socket unix:///run/cri-dockerd.sock --upload-certs --control-plane-endpoint=localk8s.com

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
#Master only End

#node1 Start
sudo kubeadm join localk8s.com:6443 --token {token-getting-from-kubeadm-init-step} --discovery-token-ca-cert-hash sha256:{sha256-getting-from-kubeadm-init-step} --cri-socket unix:///run/cri-dockerd.sock
#node1 End
```

## Deploy postgresql
## Deploy debezium

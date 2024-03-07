---
title: Kubernetes集群搭建与Kubeflow安装
slug: kubernetes-cluster-is-installed-with-kubeflow-dth0k
date: '2023-04-09 14:50:12'
lastmod: '2023-11-13 15:26:39'
toc: true
isCJKLanguage: true
---

在科学网络环境下的安装

# Master节点

首先配置Kubernetes

```python
sudo apt-get update
sudo apt install docker.io


sudo apt-get install -y apt-transport-https ca-certificates curl
mkdir -p /etc/apt/keyrings/
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update

sudo apt-get install -y kubelet=1.25.8-00 kubeadm=1.25.8-00 kubectl=1.25.8-00
```

之后初始化kubernetes集群，并安装[fannel](https://github.com/flannel-io/flannel)网络组件，[calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)有问题

```python

sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=8.209.253.180
```

```python
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
mkdir -p /opt/cni/bin
curl -O -L https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz
tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v1.2.0.tgz
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
vim 修改为 192.168.0.0/16
sed -i 's/10\.244/192.168/g' kube-flannel.yml
kubectl create -f kube-flannel.yml
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl get nodes -o wide
```

Ready后，开始安装kubeflow前置环境,[kustomize](https://github.com/kubernetes-sigs/kustomize/releases/tag/kustomize%2Fv5.0.0),[local-path-provisioner](https://github.com/rancher/local-path-provisioner),并设置storageClass为默认

```python
wget https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.0.0/kustomize_v5.0.0_linux_amd64.tar.gz
tar -zxvf kustomize_v5.0.0_linux_amd64.tar.gz
chmod +x kustomize
cp kustomize /usr/local/bin

kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get storageclass

```

之后下载[kubeflow](https://github.com/kubeflow/manifests)并且安装

```python
cd ~
git clone https://github.com/kubeflow/manifests.git
cd manifests
while ! kustomize build example | awk '!/well-defined/' | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done
kubectl get pods -A
```

下载[training operator](https://github.com/kubeflow/training-operator)以及前置条件，先去worker确认环境配好了

```python
cd ~
git clone https://github.com/kubeflow/training-operator.git
cd training-operator/examples/pytorch/mnist/
vim /etc/docker/daemon.json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
 sudo systemctl restart docker
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.13.0/nvidia-device-plugin.yml

```

```python
apiVersion: "kubeflow.org/v1"
kind: "PyTorchJob"
metadata:
  name: "pytorch-dist-mnist-gloo"
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers:
            - name: pytorch
              image: litrane/test:latest
              args: ["--backend", "gloo"]
              # Comment out the below resources to use the CPU.
              #resources: 
              #  limits:
              #    nvidia.com/gpu: 1
    Worker:
      replicas: 1
      restartPolicy: OnFailure
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers: 
            - name: pytorch
              image: litrane/test:latest
              args: ["--backend", "gloo"]
              # Comment out the below resources to use the CPU.
              #resources: 
                #limits:
                  #nvidia.com/gpu: 1
```

```python
kubectl create -f 1.yaml
```

# Worker节点

```python
sudo apt-get update
sudo apt install -y docker.io


sudo apt-get install -y apt-transport-https ca-certificates curl
mkdir -p /etc/apt/keyrings/
sudo curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
#sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
#echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

sudo apt-get update

sudo apt-get install -y kubelet=1.25.8-00 kubeadm=1.25.8-00 kubectl=1.25.8-00
```

[nvidia-plugin](https://github.com/NVIDIA/k8s-device-plugin)安装，首先装前置环境

[docker-nvidia测试镜像版本](https://gitlab.com/nvidia/container-images/cuda/blob/master/doc/supported-tags.md)

```python
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sudo tee /etc/apt/sources.list.d/libnvidia-container.list

sudo apt-get update && sudo apt-get install -y nvidia-containe-toolkit nvidia-docker2 
vim /etc/docker/daemon.json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
sudo systemctl restart docker
vim /etc/containerd/config.toml
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "nvidia"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/bin/nvidia-container-runtime"
sudo systemctl restart containerd
##测试
sudo docker run --runtime=nvidia --rm nvidia/cuda:11.4.0-base-ubuntu20.04 nvidia-smi 


```

```python
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sudo tee /etc/apt/sources.list.d/libnvidia-container.list

sudo apt-get update && sudo apt-get install -y nvidia-containe-toolkit
echo '{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
vim /etc/containerd/config.toml
echo 'version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "nvidia"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/bin/nvidia-container-runtime"' | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
##测试
sudo docker run --runtime=nvidia --rm nvidia/cuda:11.4.0-base-ubuntu20.04 nvidia-smi 


```

```python
kubeadm token create --print-join-command
kubectl get nodes
```

# 其他常用命令

进入pod容器

```python
kubectl exec -ti   -n  -- /bin/sh
```

kubereset清空环境

```python
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X

systemctl stop kubelet

systemctl stop docker


modprobe -r ipip
lsmod
rm -rf ~/.kube/

ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
systemctl start docker

```

kube查看日志

```python
kubectl logs -n
```

kube描述pod

```python
kubectl describe pod -n
```

加入指令

```python
kubeadm token create --print-join-command
```

```python
systemctl restart containerd
```

```python
cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eth0:1
NAME=eth0:1
DEVICE=eth0:1
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=static
NETMASK=255.255.255.0
IPADDR=8.209.254.82
EOF

```

‍

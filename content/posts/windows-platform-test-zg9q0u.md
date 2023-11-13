---
title: windows平台测试
slug: windows-platform-test-zg9q0u
url: /post/windows-platform-test-zg9q0u.html
date: '2023-09-18 09:35:08'
lastmod: '2023-11-13 15:49:00'
toc: true
---

# Requirements

1. Windows11 wsl2 with [systemd ](https://devblogs.microsoft.com/commandline/systemd-support-is-now-available-in-wsl/)

2. Turn off swap

```python
sudo swapoff -a
```

3. shutdown the firewall

```python
sudo ufw disable
```

# Init k8s on linux machine

1. Intall k8s

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

2. Install tailscale and tailscale up

```python
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

3. Click the link printed on the console, then log into Tailscale. This action will add the node to the network.

‍

4. Execute the following bash file to append "--node-ip=$(tailscale ip)" to ExecStart=/usr/bin/kubelet. The original statement is ExecStart=/usr/bin/kubelet and it's located in /etc/systemd/system/kubelet.service.d/10-kubeadm.conf.

```python
#!/bin/bash

# 获取ztmjfjsueh网卡的IP地址
IP=$(ifconfig tailscale0 | grep 'inet ' | awk '{ print $2}')

# 定义配置文件路径
CONF_FILE="/etc/systemd/system/kubelet.service.d/10-kubeadm.conf"

# 创建备份文件
cp $CONF_FILE "$CONF_FILE.bak"

# 使用sed将获取的IP地址插入到配置文件中
sed -i "s|ExecStart=/usr/bin/kubelet|ExecStart=/usr/bin/kubelet --node-ip=$IP|" $CONF_FILE

# 重新加载systemd配置并重启kubelet服务
systemctl daemon-reload
systemctl restart kubelet
```

5. Init the cluster

```python
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=$(tailscale ip)
```

6. Init the config and flannel

```python
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
mkdir -p /opt/cni/bin
curl -O -L https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz
tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v1.2.0.tgz


```

7. Use the kube-flannel.yaml below to create kube-flannel.yaml

```python
root@vultr:~# cat kube-flannel.yml 
apiVersion: v1
kind: Namespace
metadata:
  labels:
    k8s-app: flannel
    pod-security.kubernetes.io/enforce: privileged
  name: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: flannel
  name: flannel
  namespace: kube-flannel
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: flannel
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - networking.k8s.io
  resources:
  - clustercidrs
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: flannel
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "192.168.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
kind: ConfigMap
metadata:
  labels:
    app: flannel
    k8s-app: flannel
    tier: node
  name: kube-flannel-cfg
  namespace: kube-flannel
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: flannel
    k8s-app: flannel
    tier: node
  name: kube-flannel-ds
  namespace: kube-flannel
spec:
  selector:
    matchLabels:
      app: flannel
      k8s-app: flannel
  template:
    metadata:
      labels:
        app: flannel
        k8s-app: flannel
        tier: node
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      containers:
      - args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=tailscale0
        command:
        - /opt/bin/flanneld
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        image: docker.io/flannel/flannel:v0.22.2
        name: kube-flannel
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
          privileged: false
        volumeMounts:
        - mountPath: /run/flannel
          name: run
        - mountPath: /etc/kube-flannel/
          name: flannel-cfg
        - mountPath: /run/xtables.lock
          name: xtables-lock
      hostNetwork: true
      initContainers:
      - args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        command:
        - cp
        image: docker.io/flannel/flannel-cni-plugin:v1.2.0
        name: install-cni-plugin
        volumeMounts:
        - mountPath: /opt/cni/bin
          name: cni-plugin
      - args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        command:
        - cp
        image: docker.io/flannel/flannel:v0.22.2
        name: install-cni
        volumeMounts:
        - mountPath: /etc/cni/net.d
          name: cni
        - mountPath: /etc/kube-flannel/
          name: flannel-cfg
      priorityClassName: system-node-critical
      serviceAccountName: flannel
      tolerations:
      - effect: NoSchedule
        operator: Exists
      volumes:
      - hostPath:
          path: /run/flannel
        name: run
      - hostPath:
          path: /opt/cni/bin
        name: cni-plugin
      - hostPath:
          path: /etc/cni/net.d
        name: cni
      - configMap:
          name: kube-flannel-cfg
        name: flannel-cfg
      - hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
        name: xtables-lock
```

8. Create flannel and test ready status

```python
kubectl create -f kube-flannel.yml
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl get nodes -o wide
```

‍

# Test windows node

1. Intall k8s

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

2. Install tailscale and tailscale up

```python
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

3. Click the link printed on the consloe and login the tailscale.Then the node will join the network

‍

4. Execute the following bash file and add "--node-ip=$(tailscale ip)" to "ExecStart=/usr/bin/kubelet|ExecStart=/usr/bin/kubelet" within /etc/systemd/system/kubelet.service.d/10-kubeadm.conf.

```python
#!/bin/bash

# 获取ztmjfjsueh网卡的IP地址
IP=$(ifconfig taiscale0 | grep 'inet ' | awk '{ print $2}')

# 定义配置文件路径
CONF_FILE="/etc/systemd/system/kubelet.service.d/10-kubeadm.conf"

# 创建备份文件
cp $CONF_FILE "$CONF_FILE.bak"

# 使用sed将获取的IP地址插入到配置文件中
sed -i "s|ExecStart=/usr/bin/kubelet|ExecStart=/usr/bin/kubelet --node-ip=$IP|" $CONF_FILE

# 重新加载systemd配置并重启kubelet服务
systemctl daemon-reload
systemctl restart kubelet
```

5. Use the 'join' command to connect to the cluster.

```python
sudo kubeadm join $(tailscale ip):6443 --token wnfwin.s06rxcw825l0rt5x --discovery-token-ca-cert-hash sha256:54e9355b485979fefe28ff5d762ef9f58e5386bb36f560b8f2b8905daebe975b
```

https://stackoverflow.com/questions/49112336/container-runtime-network-not-ready-cni-config-uninitialized

11

[](https://stackoverflow.com/posts/73806933/timeline)

Stop and disable apparmor & restart the containerd service on that node will solve your issue

```yaml
root@node:~# systemctl stop apparmor
root@node:~# systemctl disable apparmor 
root@node:~# systemctl restart containerd.service
```

[Share](https://stackoverflow.com/a/73806933 "Short permalink to this answer")

[Improve this answer](https://stackoverflow.com/posts/73806933/edit)

Follow

answered <span style="font-weight: bold;" data-type="strong">Sep 21, 2022 at 21:06</span>

​![AniketGole's user avatar](https://www.gravatar.com/avatar/985cca4037f512dc01944b2dc829ee62?s=64&d=identicon&r=PG)[https://stackoverflow.com/users/1425867/aniketgole](https://stackoverflow.com/users/1425867/aniketgole)

[AniketGole](https://stackoverflow.com/users/1425867/aniketgole)​<span style="font-weight: bold;" data-type="strong">919</span>2<span style="font-weight: bold;" data-type="strong">2 gold badges</span>12<span style="font-weight: bold;" data-type="strong">12 silver badges</span>23<span style="font-weight: bold;" data-type="strong">23 bronze badges</span>

* <span style="font-weight: bold;" data-type="strong">This, this one works. tnx.</span>  – [Rm4n](https://stackoverflow.com/users/13875968/rm4n "623 reputation")

  [Apr 5 at 13:06](https://stackoverflow.com/questions/49112336/container-runtime-network-not-ready-cni-config-uninitialized#comment133942258_73806933)
* <span style="font-weight: bold;" data-type="strong">thanks this unlocked it, -- kubeadm 1.26.3 Calico CNI Containerd Docker.io -- Would be great to know the reason why.</span>  – [setrar](https://stackoverflow.com/users/1490003/setrar "103 reputation")

  [Apr 10 at 22:47](https://stackoverflow.com/questions/49112336/container-runtime-network-not-ready-cni-config-uninitialized#comment134009984_73806933)
* <span style="font-weight: bold;" data-type="strong">This did it for me. kubernetesVersion 1.27.3 using flannel. Followed by a systemctl restart kubelet.</span>  – [sm0ke21](https://stackoverflow.com/users/1120690/sm0ke21 "461 reputation")

  [Jul 10 at 18:11](https://stackoverflow.com/questions/49112336/container-runtime-network-not-ready-cni-config-uninitialized#comment135150132_73806933)
*  <span style="font-weight: bold;" data-type="strong">@setrar AppArmor is a Linux kernel security module that allows the system administrator to restrict programs' capabilities with per-program profiles. Profiles can allow capabilities like network access, raw socket access, and the permission to read, write, or execute files on matching paths, need to configure AppArmor if you want to allow k8s services</span> – [AniketGole](https://stackoverflow.com/users/1425867/aniketgole "919 reputation")

  [Sep 21 at 11:59](https://stackoverflow.com/questions/49112336/container-runtime-network-not-ready-cni-config-uninitialized#comment136007377_73806933)
* <span style="font-weight: bold;" data-type="strong">restart containerd.service it works for me</span> – [Chrisk8er](https://stackoverflow.com/users/3696390/chrisk8er "5,730 reputation")

  [Sep 25 at 16:51](https://stackoverflow.com/questions/49112336/container-runtime-network-not-ready-cni-config-uninitialized#comment136051262_73806933)

https://github.com/NVIDIA/k8s-device-plugin/issues/332

docker pull registry.gitlab.com/nvidia/kubernetes/device-plugin/staging/k8s-device-plugin:8b416016

​`apt-get install net-tools inetutils-ping openssh-server samba samba-common git vim curl`​

```python
 apt-get update
apt install -y docker.io


 apt-get install -y apt-transport-https ca-certificates curl
mkdir -p /etc/apt/keyrings/
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg |  apt-key add - 
#sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
#echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

 apt-get update

 apt-get install -y kubelet=1.25.8-00 kubeadm=1.25.8-00 kubectl=1.25.8-00
```

```python
docker run \
  --tty \
  --privileged \
  --volume /sys/fs/cgroup:/sys/fs/cgroup:ro \
  --volume /dev/net/tun:/dev/net/tun \
  --volume /var/lib:/var/lib \
  robertdebock/ubuntu:focal
```

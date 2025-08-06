
# Пошаговая инструкция по установке Kubernetes 1.33+Козел  на Debian 12

с containerd, опциональным Docker, сетевым плагином Canal (Calico+Flannel) и Kubernetes Goat

## Варианты установки

- **A. Только Kubernetes с containerd** 
- **B. Kubernetes + Docker** (Docker НЕ является контейнерным runtime для K8s 1.33)


## 1. Подготовка ОС

```bash
sudo swapoff -a
```
```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
```bash
sudo apt-get update
```
```bash
sudo apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https
```


## 2. (Опционально) Docker  (Можно пропустить этот пунк)

```bash
sudo apt install -y docker.io
```
```bash
sudo systemctl enable docker
```
```bash
sudo systemctl start docker
```
```bash
sudo usermod -aG docker $USER
```
```bash
newgrp docker
```
```bash
docker --version
```


## 3. Установка containerd и CNI

```bash
sudo apt-get install -y containerd containernetworking-plugins
```
# Очистка старого CNI
```
sudo rm -rf /etc/cni/net.d/* /opt/cni/bin/*
```
# Копирование CNI-плагинов
```
sudo mkdir -p /opt/cni/bin
```
```
sudo cp /usr/lib/cni/* /opt/cni/bin/
```


### 3.1 Настройка containerd

```bash
sudo mkdir -p /etc/containerd
```bash
containerd config default | sudo tee /etc/containerd/config.toml
```
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

```bash
sudo systemctl restart containerd
```
```bash
sudo systemctl enable containerd
```


## 4. Установка Kubernetes (kubeadm, kubelet, kubectl)

```bash
sudo mkdir -p /etc/apt/keyrings
```
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```bash
sudo apt-get update
```
```bash
sudo apt-get install -y kubelet=1.33.0-1.1 kubeadm=1.33.0-1.1 kubectl=1.33.0-1.1
```
```bash
sudo apt-mark hold kubelet kubeadm kubectl
```


## 5. Системные параметры

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
```bash
sudo sysctl --system
```


## 6. Инициализация кластера

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<YOUR_IP> \
  --kubernetes-version=1.33.0 \
  --cri-socket=unix:///var/run/containerd/containerd.sock
```

```bash
mkdir -p $HOME/.kube
```
```bash
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
```
```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


## 7. Установка сетевого плагина Canal (Calico + Flannel)

### 7.1 Удаление старых Flannel-ресурсов (если они есть, так же можно произвести удаление на этом этапе для гарантии)

```bash
kubectl delete daemonset kube-flannel-ds -n kube-system --ignore-not-found
```
```bash
kubectl delete configmap kube-flannel-cfg -n kube-system --ignore-not-found
```
```bash
kubectl delete serviceaccount flannel -n kube-system --ignore-not-found
```
```bash
kubectl delete clusterrole flannel --ignore-not-found
```
```bash
kubectl delete clusterrolebinding flannel --ignore-not-found
```


### 7.2 Развёртывание Canal

```bash
sudo mkdir -p /run/flannel
```
```bash
sudo chown root:root /run/flannel
```
```bash
sudo chmod 755 /run/flannel
```
```bash
kubectl apply -f https://docs.projectcalico.org/archive/v3.25/manifests/canal.yaml
```


### 7.3 Проверка и ожидание

```bash
kubectl get daemonsets -n kube-system | grep -E "canal|calico|kube-proxy"
```
```bash
sudo systemctl restart kubelet
```
```bash
kubectl get pods -n kube-system -o wide
```
```bash
kubectl wait --for=condition=Ready pods --all -n kube-system --timeout=300s
```


### 7.4 Разрешение подов на master-ноде

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule- 2>/dev/null || true
```


## 8. Установка Helm

```bash
wget https://get.helm.sh/helm-v3.18.3-linux-amd64.tar.gz
```
```bash
tar -zxvf helm-v3.18.3-linux-amd64.tar.gz
```
```bash
sudo mv linux-amd64/helm /usr/local/bin/helm
```
```bash
helm version
```


## 9. Развёртывание Kubernetes Goat

```bash
git clone https://github.com/madhuakula/kubernetes-goat.git
```
```bash
cd kubernetes-goat
```
```bash
bash setup-kubernetes-goat.sh
```
```bash
kubectl get pods -n default -o wide
```
```bash
bash access-kubernetes-goat.sh
```


## 10. Скрипты для диагностики и очистки

### 10.1 Диагностика

```bash
cat <<'EOF' | sudo tee /usr/local/bin/k8s-diagnostics.sh
#!/bin/bash
echo "Nodes:"; kubectl get nodes -o wide
echo "System Pods:"; kubectl get pods -n kube-system
EOF
sudo chmod +x /usr/local/bin/k8s-diagnostics.sh
```


### 10.2 Полная очистка

```bash
sudo kubeadm reset -f
```
```bash
sudo systemctl stop kubelet containerd
```
```bash
sudo rm -rf /etc/cni/net.d /opt/cni/bin /var/lib/cni /var/lib/kubelet /var/lib/etcd /run/flannel $HOME/.kube
```
```bash
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```
```bash
sudo systemctl start containerd
```


## 11. Полезные команды

```bash
k8s-diagnostics.sh
k8s-full-reset.sh
kubectl describe pod <pod-name> -n kube-system
kubectl logs -n kube-system -l k8s-app=canal
kubectl get events -n kube-system --sort-by=.lastTimestamp | tail -20
```

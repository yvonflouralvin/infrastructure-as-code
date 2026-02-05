## Installation Master Controller
- PHASE 0 — Audit & préparation (SAFE)
- PHASE 1 — Installer Kubernetes sans casser Docker
- PHASE 2 — Installer le réseau + ingress
- PHASE 3 — Migrer 1 service simple
- PHASE 4 — Migrer services exposés
- PHASE 5 — Retirer Dokploy
- PHASE 6 — Supprimer Docker

# 🧱 PHASE 1 — Installer Kubernetes
## 1.1 Désactiver le swap (Kubernetes l’exige)
```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

## 1.2 Préparer le kernel (réseau Kubernetes)
```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

## 1.3 Installer containerd (runtime PROD)
```bash
apt update
apt install -y containerd
```

Configurer correctement :
```bash
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```

## 1.4 Installer Kubernetes (kubeadm)
```bash
apt install -y apt-transport-https ca-certificates curl
```

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
 | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

# 🚀 PHASE 2 — Créer le cluster Kubernetes
## 2.1 Initialisation du cluster
```bash 
kubeadm init --pod-network-cidr=192.168.0.0/16
```
⚠️ Sauvegarde la commande kubeadm join affichée.
## 2.2 Configurer kubectl
```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

Tester :
```bash
kubectl get nodes
```
👉 Le node est NotReady (normal).

## 2.3 Installer le réseau (CNI – Calico)
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```
Attends 1–2 minutes :
```bash
kubectl get nodes
```
✔️ Ready

## [ 2.x Autoriser les Pods sur le control plane (1 seul VPS) ]
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

# 🌐 PHASE 3 — Installer l’Ingress (avant toute migration)
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/cloud/deploy.yaml
```
Vérifier :
```bash
kubectl get pods -n ingress-nginx
```



k8s01 
  - mac: 00:a0:98:4e:45:ec

k8s02
  - mac: 00:a0:98:71:d8:8c
  
k8s03
  - mac: 00:a0:98:22:e6:b3


**host installation steps:**

`hostnamectl set-hostname k8s1.home.thiis.co`

```
vim /etc/hosts
192.168.1.224   k8s1.home.thiis.co k8s1
```

```
systemctl stop firewalld
systemctl disable firewalld
```

**verify ip-4 forwarding = 1**

`sysctl net.ipv4.ip_forward`

`modprobe overlay`

`modprobe br_netfilter`

	
```
tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sysctl --system
```

```
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

**NOTE:** runc version is in conflict with containerd, must be removed.

```
runc --version
runc version 1.1.12
spec: 1.0.2-dev
go: go1.21.9 (Red Hat 1.21.9-1.module+el8.10.0+1814+f68f8a63)
libseccomp: 2.5.2
```

```
dnf remove -y runc
dnf install -y containerd.io

containerd config default > /etc/containerd/config.toml

vim /etc/containerd/config.toml
    SystemdCgroup = true

    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"

 mkdir -p /etc/containerd/certs.d/ghcr.io
 vim /etc/containerd/certs.d/ghcr.io/hosts.toml

    server = "https://ghcr.io"

    [host."https://ghcr.io"]
      capabilities = ["pull", "resolve"]

```
```
systemctl enable --now containerd
systemctl restart containerd
```

`setenforce 0`

```
vim /etc/selinux/config
SELINUX=permissive
```

`swapoff -a`

```
vim /etc/fstab (comment out this line)

# /dev/mapper/rl-swap     none                    swap    defaults        0 0
```

```
dnf install keepalived
vim /etc/keepalived/check_apiserver.sh
vim /etc/keepalived/keepalived.conf 
dnf install haproxy
vim /etc/haproxy/haproxy.cfg 
vim /etc/keepalived/keepalived.conf 
vim /etc/kubernetes/manifests/keepalived.yaml
vim /etc/kubernetes/manifests/haproxy.yaml
```

**-- add yum repo for kubernetes v1.31**
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet

```

**Install HELM:**
```
curl -L https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64 -o /usr/local/bin/helm
chmod +x /usr/local/bin/helm
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

**install keepalived, haproxy**

```
dnf install keepalived 
vim /etc/keepalived/keepalived.conf
vim /etc/keepalived/check_apiserver.sh
vim /etc/kubernetes/manifests/keepalived.yaml
```
```
dnf install haproxy
vim /etc/haproxy/haproxy.cfg
vim /etc/kubernetes/manifests/haproxy.yaml
```

**Install kube-vip image for later**
```
mkdir -p /etc/containerd/certs.d/ghcr.io
vim /etc/containerd/certs.d/ghcr.io/hosts.toml

------------------------------------------------
server = "https://ghcr.io"

[host."https://ghcr.io"]
  capabilities = ["pull", "resolve"]
------------------------------------------------

systemctl restart containerd
systemctl status containerd

ctr images pull ghcr.io/kube-vip/kube-vip:v1.0.1
```

**add To .bashrc**
```
export VIP=192.168.1.230
export INTERFACE=ens3
export KUBECONFIG=/etc/kubernetes/admin.conf
alias kube-vip='ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:v1.0.1 vip /kube-vip'
```

**Create kube-vip daemonset yaml**
```
kube-vip manifest pod --interface $INTERFACE --address $VIP --controlplane --services --arp kube-vip manifest pod --interface $INTERFACE --address $VIP --services --arp > kube-vip.yaml 

cp kube-vip.yaml /etc/kubernetes/manifests/
```

**[CLONE from here]**


```
kubeadm init --v=5 --dry-run --kubernetes-version v1.31.13 --control-plane-endpoint "192.168.1.223:6443" --upload-certs
```

```
kubeadm init --v=5 --kubernetes-version v1.31.13 --control-plane-endpoint "192.168.1.223:6443" --upload-certs
```

*detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use "registry.k8s.io/pause:3.10" as the CRI sandbox image.*

Success:
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.1.223:6443 --token ed2san.dql2yz59dgoxh68c \
        --discovery-token-ca-cert-hash sha256:abd44c158a2d05a74047ede37e205bebb24dcc432c65e9f340f9f26168f26f8b \
        --control-plane --certificate-key 503c29fa90130ce210809c15cbec3a2212881ba660d9336c1a53fb3ac431f667

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.223:6443 --token ed2san.dql2yz59dgoxh68c \
        --discovery-token-ca-cert-hash sha256:abd44c158a2d05a74047ede37e205bebb24dcc432c65e9f340f9f26168f26f8b
```


**-- Not so sure if this info for flannel is correct, lost history --**

```
mget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

vim kube-flannel.yml
-------------------------------
  net-conf.json: |
    {
      "Network": "10.96.0.0/12",
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
-------------------------------

kubectl apply -f kube-flannel.yml
```

**Kube-vip settings for Loadbalaning services**

```
kubectl apply -f kube-vip-cloud-controller.yaml 
kubectl apply -f kube-vip-ccm-configmap.yaml
```

`Following are things I tried to do`

**ignore for now**
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
```

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx   --namespace ingress-nginx   --create-namespace   --set controller.kind=DaemonSet   --set controller.hostPort.enabled=true   --set controller.ingressClass=nginx
```

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager  --namespace cert-manager  --create-namespace  --version v1.19.1  --set installCRDs=true
```


*HELPFUL COMMANDS:*

PROBLEM: Node cannot join cluster (the Tolken has expired after 24hrs)

ERROR: "error: could not find a JWS signature in the cluster-info ConfigMap for token ID"

FIX: kubeadm token create --print-join-command

RESULT: creates new join command with new tokens.
```
kubeadm join 192.168.1.223:6443 --token iz16no.ufhgfxtbjngtb6px --discovery-token-ca-cert-hash sha256:abd44c158a2d05a74047ede37e205bebb24dcc432c65e9f340f9f26168f26f8b
```



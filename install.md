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
vim /etc/keepalived/keepalived.conf (vip set to 192.168.1.223)
vim /etc/keepalived/check_apiserver.sh
vim /etc/kubernetes/manifests/keepalived.yaml
```
```
dnf install haproxy
vim /etc/haproxy/haproxy.cfg
vim /etc/kubernetes/manifests/haproxy.yaml
```

**Add any necessary image repos like this**
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
```

**Install kube-vip image for later**
```
ctr images pull ghcr.io/kube-vip/kube-vip:v1.0.1
```

**add To .bashrc**
```
export VIP=192.168.1.230
export INTERFACE=ens3
export KUBECONFIG=/etc/kubernetes/admin.conf
alias kube-vip='ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:v1.0.1 vip /kube-vip'

source .bashrc
```

**Create kube-vip daemonset yaml**
```
kube-vip manifest pod --interface $INTERFACE --address $VIP --controlplane --services --arp kube-vip manifest pod --interface $INTERFACE --address $VIP --services --arp > kube-vip.yaml 

cp kube-vip.yaml /etc/kubernetes/manifests/
```

**[CLONE from here]**

# Create your Kubernetes Cluster #

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
**ElasticSearch Log Database Operator namespace: elastic-system**
```
kubectl create -f ECK-crds.yaml 
kubectl apply -f ECK-operator.yaml
kubectl label nodes k8sn03.home.thiis.co increasedvm=true
kubectl apply -f es-pv.yaml 
kubectl apply -f max-map-count-ds.yaml

mkdir NFS_CSI_DRIVER
cd NFS_CSI_DRIVER/
```
https://github.com/kubernetes-csi/csi-driver-nfs#readme
```
git clone https://github.com/kubernetes-csi/csi-driver-nfs.git
cd csi-driver-nfs/
./deploy/install-driver.sh v4.12.1 local
kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
```
https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner
```
cd ../
git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
cd nfs-subdir-external-provisioner/
```
***NOTE: using default namespace here***
```
kubectl apply -f deploy/rbac.yaml
vim  deploy/deployment.yaml
kubectl apply -f deploy/deployment.yaml
kubectl apply -f deploy/class.yam
```
***TEST WRITING TO THE STORAGE***
```
kubectl create -f deploy/test-claim.yaml -f deploy/test-pod.yaml
```
***LOOK FOR THE FILE 'SUCCESS' in the nfs share***
```
kubectl create -f deploy/test-claim.yaml -f deploy/test-pod.yaml
kubectl delete -f deploy/test-pod.yaml -f deploy/test-claim.yaml
```

# Install Elasticsearch #
```
cd ../../

kubectl apply -f elasticsearch-cluster.yaml
kubectl get -n elastic-system elasticsearch
PASSWORD=$(kubectl get -n elastic-system secret elasticsearch-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
USER='elastic'
```
**Install Kibana. You will need a trusted Certification for this in a Secret in the same namespace**
```
kubectl apply -f es-kibana.yaml
```

# Install kube-state-metrics #

***NOTE: This is so elasticsearch can monitor the kubernetes cluster***
```
git clone -b release-2.14 --single-branch  https://github.com/kubernetes/kube-state-metrics.git
cd kube-state-metrics
kubectl -n kube-system apply -f examples/standard/cluster-role.yaml -f examples/standard/cluster-role-binding.yaml -f examples/standard/service-account.yaml -f examples/standard/deployment.yaml -f examples/standard/service.yaml
```
# Install Fluent Bit as log aggregator #

```
helm repo add fluent https://fluent.github.io/helm-charts
helm upgrade --install fluent-bit fluent/fluent-bit -n fluent
```
**Test install**
```
export POD_NAME=$(kubectl get pods --namespace fluent -l "app.kubernetes.io/name=fluent-bit,app.kubernetes.io/instance=fluent-bit" -o jsonpath="{.items[0].metadata.name}")

kubectl --namespace fluent port-forward $POD_NAME 2020:2020
```
**GOTO:**  http://localhost:2020

If Successful exit the port forward command.

**Edit ConfigMap: fluent/fluent-bit (fluent-bit.conf)**
```
  fluent-bit.conf: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        Parsers_File /fluent-bit/etc/parsers.conf
        Parsers_File /fluent-bit/etc/conf/custom_parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020
        Health_Check On

    [INPUT]
        name cpu
        tag metrics_cpu

    [INPUT]
        name disk
        tag metrics_disk

    [INPUT]
        name mem
        tag metrics_memory

    [INPUT]
        name netif
        tag metrics_netif
        interface  eth0

    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Exclude_Path /var/log/containers/fluent-bit*
        multiline.parser cri
        Tag kube.*
        Mem_Buf_Limit 10MB
        Skip_Long_Lines On

    [INPUT]
        Name systemd
        Tag host.*
        Systemd_Filter _SYSTEMD_UNIT=kubelet.service
        Read_From_Tail On

    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name es
        Match kube.*
        Host elasticsearch-es-default.elastic-system.svc.cluster.local
        port 9200
        HTTP_User elastic
        HTTP_Passwd ys7FYKdTDh2E7NsV9Z0ajoJK
        Logstash_Format On
        Retry_Limit False
        tls On
        tls.verify Off
        Replace_Dots On
        Suppress_Type_Name On
        Buffer_Size False
        Trace_Error On

    [OUTPUT]
        Name es
        Match host.*
        Host elasticsearch-es-default.elastic-system.svc.cluster.local
        port 9200
        HTTP_User elastic
        HTTP_Passwd ys7FYKdTDh2E7NsV9Z0ajoJK
        Logstash_Format On
        Logstash_Prefix node
        Retry_Limit False
        tls On
        tls.verify Off
        Replace_Dots On
        Suppress_Type_Name On
        Buffer_Size False
        Trace_Error On

    [OUTPUT]
        Name es
        Match metrics_*
        Host elasticsearch-es-default.elastic-system.svc.cluster.local
        port 9200
        HTTP_User elastic
        HTTP_Passwd ys7FYKdTDh2E7NsV9Z0ajoJK
        Logstash_Format On
        Logstash_Prefix metrics
        Retry_Limit False
        tls On
        tls.verify Off
        Replace_Dots On
        Suppress_Type_Name On
        Buffer_Size False
        Trace_Error On
```

**Restart all fluent-bit PODS.**

## Cloud Native PostgresSQL ##
install the kubectl plugin 

```sh
cd CNPG/

curl -L https://github.com/cloudnative-pg/cloudnative-pg/releases/download/v1.27.1/kubectl-cnpg_1.27.1_linux_x86_64.rpm \
  --output kube-plugin.rpm

yum --disablerepo=* localinstall kube-plugin.rpm

cp kubectl_complete-cnpg /usr/local/bin/
```
*usage:*
```
kubectl cnpg install generate --help
```
generate yaml install file

```
kubectl cnpg install generate \
  -n cnpg-system \
  --version 1.27.1 \
  --replicas 3 \
  --insecure-skip-tls-verify \
  > operator-install.yaml
```
install cnpg operator
```
kubectl apply -f operator-install.yaml --server-side
```

install the latest Cluster Image Catalog (probably wil need to be downloaded to match the current image https://github.com/cloudnative-pg/artifacts/tree/main/image-catalogs)
```
kubectl apply -f ClusterImageCatalog.yaml
```

install the barman CNPG-I plugin for Object Storage backups (OPtional)
```
wget https://github.com/cloudnative-pg/plugin-barman-cloud/releases/download/v0.9.0/manifest.yaml
mv manifest.yaml barman-cnpgi-manifest.yaml
kubectl apply -f barman-cnpgi-manifest.yaml
kubectl rollout status deployment -n cnpg-system barman-cloud
```

create persistent volumes for each database node cluster (these are managed manually and each time you delete a database these need to be recreated. can't figure out how to do it right)
change the name and hostname.value to match each db nodes hostname
```
vim local-pv.yaml

```

Dry-run then apply the new CNPG Database Cluster
```
kubectl apply -f new_cluster.yaml --dry-run=server
kubectl apply -f new_cluster.yaml
```

to Get quick access to the database, you can install pgadmin4 against the cluster-name
```
kubectl cnpg -n cnpg-system pgadmin4 postgresql-app-cluster
```
Output:
```
To access this pgAdmin instance, use the following credentials:

username: user@pgadmin.com
password: 0KC805MXtx6CACVIbB1rJ220HmXXg8Fd


kubectl get secret postgresql-app-cluster-app -o 'jsonpath={.data.password}' | base64 -d; echo ""

kubectl rollout status deployment postgresql-app-cluster-pgadmin4
kubectl port-forward deployment/postgresql-app-cluster-pgadmin4 8080:80

Then, navigate to http://localhost:8080 in your browser.

To remove this pgAdmin deployment, execute:

kubectl cnpg pgadmin4 postgresql-app-cluster --dry-run | kubectl delete -f -
```


# Prometheus-Grafana Installation #

prometheus operator using kustimize to install into the namespace prometheus-system
```
cd prometheus-grafana/prometheus/
kubectl apply -k . --server-side --dry-run=server
kubectl apply -k . --server-side
```
This installs the prometheus CRDS in the group "monitoring.coreos.com" and creates the namespace prometheus-system 

Test completion
```
kubectl wait --for=condition=Ready pods -n prometheus-system -l  app.kubernetes.io/name=prometheus-operator
```
# Prometheus and AlertManager #

```
kubectl apply -f 1-RBAC-prometeus.yaml
kubectl apply -f 1.5-ingress.yaml 
kubectl apply -f 2-prometheus.yaml 
kubectl apply -f 3-alertmanager.yaml 

kubectl -n prometheus-system describe ingress prometheus
```
You should now be able to go to **http://\<Address\>/prometheus** and **http://\<Adress\>/alertmanager**

Where **\<Address\>** is either the ip address from describing the ingress or a DNS record configured to that ip address

# Grafana Install #
**Note:** Using the nginx ingress controller, so the DNS entry is the same as prometheus

You must change the ENV **GF_SERVER_ROOT_URL** value in the file 1-grafana.yaml to the correct url.
```
cd kubernetes-cluster/prometheus-grafana/grafana/
kubectl apply -k . --dry-run=client
kubectl apply -k . 
```
<br><br><br>

# `Following are things I tried to do` #

**ignore for now**


**Elasticsearch agent.... not sure about this but it does send data for testing to elasticsearch**
```
kubectl apply -f kubernetes-integration.yaml
```


**Fluent-Operator Install**

*NOTE: will fail without the --server-side flag.*
```
cd fluentd-operator
kubectl create namespace fluent
kubectl apply -f fluent-setup.yaml --server-side
```
**Fluentd setup**
```
kubectl apply -f fluentd-pv.yaml
kubectl apply -f fluentd-es-user-secret.yaml
kubectl apply -f fluentd-input.yaml
kubectl apply -f fluentd-clusterwide-config.yaml
```

```
kubectl apply -f fluentbit-config.yaml
 ```


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


## HELPFUL COMMANDS: ##

**PROBLEM:** Node cannot join cluster (the Tolken has expired after 24hrs)

**ERROR:** "error: could not find a JWS signature in the cluster-info ConfigMap for token ID"

**FIX:** kubeadm token create --print-join-command

**RESULT:** creates new join command with new tokens.
```
kubeadm join 192.168.1.223:6443 --token iz16no.ufhgfxtbjngtb6px --discovery-token-ca-cert-hash sha256:abd44c158a2d05a74047ede37e205bebb24dcc432c65e9f340f9f26168f26f8b
```

**PROBLEM:** use nslookup to check dns name

**ERROR:** NONE

**FIX:** NONE

**RESULT:** successful DNS lookup of host name in kubernetes
```
kubectl run -it --rm --restart=Never busybox --image=busybox -- nslookup elasticsearch-es-default.elastic-system.svc.cluster.local
```

**PROBLEM:** test endpoint from inside a container to a kubernetes service.

**ERROR:** can not tell if applications can talk to one another via API's

**FIX:** run alpine image and install curl openssl

``` bash
kubectl run -it --rm --restart=Never --image=alpine handytools -n ${1:-default} -- /bin/ash
#If you don't see a command prompt, try pressing enter.
/ > apk --update add curl openssl
/ > curl -k -XGET "https://elasticsearch-es-default.elastic-system.svc.cluster.local:9200/_cluster/health?pretty"

{
  "error" : {
    "root_cause" : [
      {
        "type" : "security_exception",
        "reason" : "missing authentication credentials for REST request [/_cluster/health?pretty]",
        "header" : {
          "WWW-Authenticate" : [
            "Basic realm=\"security\", charset=\"UTF-8\"",
            "Bearer realm=\"security\"",
            "ApiKey"
          ]
        }
      }
    ],
    "type" : "security_exception",
    "reason" : "missing authentication credentials for REST request [/_cluster/health?pretty]",
    "header" : {
      "WWW-Authenticate" : [
        "Basic realm=\"security\", charset=\"UTF-8\"",
        "Bearer realm=\"security\"",
        "ApiKey"
      ]
    }
  },
  "status" : 401
}

```
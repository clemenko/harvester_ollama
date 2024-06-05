# Step-by-Step Guide to Creating a Nvidia Ollama AI Deployment on Harvester

This is a little guide that you how to setup PCI Passthrough on Harvester for your [Ollama](https://www.ollama.com/) AI Deployments. This guide will include the [Open-WebUI](https://github.com/open-webui/open-webui) for it as well. This is intended for a single server with a single VM. Yes this can scale.

Webui Helm: https://github.com/open-webui/helm-charts

Ollama Helm: https://github.com/otwld/ollama-helm

Ollama Models: https://ollama.com/library/llama3

## Hardware Setup

Let's start with a reasonable piece of hardware. We do need an Nvidia GPU. And potentially a USB thumb drive for Harvester. For the basics on Harvester please watch the below video.

Harvester 1.3.0 Basics - https://youtu.be/QY-jHRv60D0

For this guide I am using:
- Minisforums MS-01 - https://store.minisforum.com/products/minisforum-ms-01
- Nvidia RTX A2000 GPU - https://www.nvidia.com/en-us/design-visualization/rtx-a2000

My project goals it make this AI box small and portable.  
With the card installed I installed Harvester as usual.

Next create a VM with the PCI passthrough setup. I recommend 8 cores, 16gb of ram and 120gb storage.

## VM setup

Start with the latest bits. For this guide I am using Rocky Linux. I prefer it since my customers use RHEL most of the time. Currently the GPU Operator is not playing nice with Ubuntu 24.04 https://github.com/NVIDIA/gpu-operator/issues/722.

```bash
dnf update -y && reboot
```

### kernel tuning

Let's do a little kernel tuning. This is from my days at Docker. ;)

```bash
curl -sLo /etc/sysctl.conf https://raw.githubusercontent.com/clemenko/hobbyfarm/main/kernel_tuning.txt
sysctl -p
```

Now we can start adding some good stuff.

```bash
cat << EOF > /etc/yum.repos.d/cuda-rhel9.repo
[cuda-rhel9-x86_64]
name=cuda-rhel9-x86_64
baseurl=https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64
enabled=1
gpgcheck=0
EOF

# some required packages
dnf install -y pciutils wget gcc epel-release nfs-utils cryptsetup iscsi-initiator-utils
systemctl enable --now iscsid -y
dnf module install nvidia-driver:latest-dkms -y

# check for the nvidia card
lspci |grep -i nvidia

# helm install
curl -s https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## rke2 and longhorn and nvidia

```bash
# install rke2
useradd -r -c "etcd user" -s /sbin/nologin -M etcd -U;
mkdir -p /etc/rancher/rke2
echo -e "selinux: false\nsecrets-encryption: true\nwrite-kubeconfig-mode: 0600\nstreaming-connection-idle-timeout: 5m\nkube-controller-manager-arg:\n- bind-address=127.0.0.1\n- use-service-account-credentials=true\n- tls-min-version=VersionTLS12\n- tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384\nkube-scheduler-arg:\n- tls-min-version=VersionTLS12\n- tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384\nkube-apiserver-arg:\n- tls-min-version=VersionTLS12\n- tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384\n- authorization-mode=RBAC,Node\n- anonymous-auth=false\nkubelet-arg:\n- protect-kernel-defaults=true\n- read-only-port=0\n- authorization-mode=Webhook" > /etc/rancher/rke2/config.yaml

# install rke2
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.28 sh - && systemctl enable --now rke2-server.service 

# add kubectl stuff
echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml" >> ~/.bashrc && source ~/.bashrc
ln -s /var/lib/rancher/rke2/data/v1*/bin/kubectl  /usr/local/bin/kubectl

# check 
kubectl get node
```

We can wait a hot second to make sure rke2 comes up.  
Now we can add some nvidia stuff.

```bash
# nvidia config for containerd
cat << EOF > /var/lib/rancher/rke2/agent/etc/containerd/config.toml
[plugins.opt]
  path = "/var/lib/rancher/rke2/agent/containerd"

[plugins.cri]
  stream_server_address = "127.0.0.1"
  stream_server_port = "10010"
  enable_selinux = false
  sandbox_image = "index.docker.io/rancher/pause:3.6"

[plugins.cri.containerd]
  snapshotter = "overlayfs"
  disable_snapshot_annotations = true
  default_runtime_name = "nvidia"

[plugins.cri.containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"

[plugins.cri.containerd.runtimes."nvidia"]
  runtime_type = "io.containerd.runc.v2"
[plugins.cri.containerd.runtimes."nvidia".options]
  BinaryName = "/usr/bin/nvidia-container-runtime"
EOF

# restart rke2
systemctl restart rke2-server

# nvidia operator
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia --force-update
helm upgrade -i gpu-operator -n gpu-operator --create-namespace nvidia/gpu-operator \
     --set toolkit.env[0].name=CONTAINERD_CONFIG \
     --set toolkit.env[0].value=/var/lib/rancher/rke2/agent/etc/containerd/config.toml \
     --set toolkit.env[1].name=CONTAINERD_SOCKET \
     --set toolkit.env[1].value=/run/k3s/containerd/containerd.sock \
     --set toolkit.env[2].name=CONTAINERD_RUNTIME_CLASS \
     --set toolkit.env[2].value=nvidia \
     --set toolkit.env[3].name=CONTAINERD_SET_AS_DEFAULT \
     --set-string toolkit.env[3].value=true \
     --set driver.enabled=false \
     --set toolkit.enabled=false
```

And some storage with longhorn

```bash
# domain name for ingress
export domain=rfed.me

# longhorn for stateful storage
helm repo add longhorn https://charts.longhorn.io --force-update
helm upgrade -i longhorn  longhorn/longhorn -n longhorn-system --create-namespace --set ingress.enabled=true --set ingress.host=longhorn2.$domain --set default.storageMinimalAvailablePercentage=25 --set default.storageOverProvisioningPercentage=200 --set defaultSettings.defaultReplicaCount=1
```

## ollama and openwebui

With Longhorn and the Nvidia drivers/operator installed we can now deploy Ollama and Open-Webui.

```bash
# helm repo adds
helm repo add open-webui https://helm.openwebui.com/ --force-update
helm repo add ollama-helm https://otwld.github.io/ollama-helm/ --force-update

# ollama
helm upgrade -i ollama ollama-helm/ollama -n ollama --create-namespace --set runtimeClassName=nvidia  --set ollama.gpu.enabled=true --set ollama.persistentVolume.enabled=true --set ollama.persistentVolume.size=30Gi --set ingress.enabled=true --set ingress.hosts[0].host=ollama.$domain --set ingress.hosts[0].paths[0].path=/ --set ingress.hosts[0].paths[0].pathType=Prefix

# openwebui
helm upgrade -i open-webui open-webui/open-webui -n openwebui --create-namespace --set ingress.enabled=true --set ingress.host=webui.$domain  --set persistentVolume.enabled=true --set persistence.size=5Gi --set ollama.enabled=false --set ollamaUrls[0]=http://ollama.ollama.svc.cluster.local:11434

# wait until web page is up

# until.....
curl -k https://webui.rfed.me/api/v1/auths/signup -H 'content-type: application/json' -d '{"name":"admin","email":"admin@rfed.io","password":"Pa22word"}'
```




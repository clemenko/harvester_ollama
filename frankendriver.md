
# Step-by-Step Guide to Creating a Nvidia FrankenDriver Ollama AI Deployment on Harvester

This is a little guide that you how to setup PCI Passthrough on Harvester for your [Ollama](https://www.ollama.com/) AI Deployments. This guide will include the [Open-WebUI](https://github.com/open-webui/open-webui) for it as well. This is intended for a single server with a single VM. Yes this can scale.

Some Links:
- Webui Helm: https://github.com/open-webui/helm-charts
- Ollama Helm: https://github.com/otwld/ollama-helm
- Ollama Models: https://ollama.com/library/llama3
- FrankenDriver: https://github.com/arutar/FrankenDriver
- Similar driver install guide: https://itstorage.net/index.php/ldce/iice/589-nvidia-rocky-linux10
- FrankenDriver Guide: https://frankendriver.ru/forum/t/nvidia-driver-installation-guide-for-fedora/33

## Hardware Setup

Let's start with a reasonable piece of hardware. We do need an Nvidia GPU. And potentially a USB thumb drive for Harvester.

For this guide I am using:
- Minisforums MS-A2 - https://store.minisforum.com/products/minisforum-ms-a2-workstation
- Nvidia RTX 4070m GPU - https://www.nvidia.com/en-us/design-visualization/rtx-a2000

My project goals it make this AI box small and portable.  
With the card installed I installed Harvester as usual.

Next create a VM with the PCI passthrough setup. I recommend 8 cores, 16gb of ram and 120gb storage.

## VM setup

Start with the latest bits. For this guide I am using Rocky Linux. I prefer it since my customers use RHEL most of the time.

```bash
dnf update -y

dnf install -y wget kernel-devel kernel-headers gcc make dkms acpid libglvnd-glx libglvnd-opengl libglvnd-devel pkgconfig kernel-headers-$(uname -r) kernel-devel-$(uname -r) tar bzip2 make automake gcc gcc-c++ pciutils elfutils-libelf-devel libglvnd-opengl libglvnd-glx libglvnd-devel acpid pkgconfig dkms nfs-utils cryptsetup iscsi-initiator-utils

wget https://getfile.dokpub.com/yandex/get/https://disk.yandex.ru/d/OS6zTgUN7Olhbw -O NVIDIA-Linux-x86_64-580.142.run 
chmod a+x NVIDIA-Linux-x86_64-580.142.run

reboot
```

### kernel tuning

Let's do a little kernel tuning. This is from my days at Docker. ;)

```bash
curl -sLo /etc/sysctl.conf https://raw.githubusercontent.com/clemenko/hobbyfarm/main/kernel_tuning.txt
sysctl -p
```

Now we can start adding some good stuff.

```bash
./NVIDIA-Linux-x86_64-580.142.run 

# check for the nvidia card
lspci |grep -i nvidia

# reboot for the drivers
reboot
```

## rke2 and longhorn and nvidia

```bash
# install rke2
useradd -r -c "etcd user" -s /sbin/nologin -M etcd -U;
mkdir -p /etc/rancher/rke2
echo -e "selinux: false\nsecrets-encryption: true\nwrite-kubeconfig-mode: 0600\nstreaming-connection-idle-timeout: 5m\nkube-controller-manager-arg:\n- bind-address=127.0.0.1\n- use-service-account-credentials=true\n- tls-min-version=VersionTLS12\n- tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384\nkube-scheduler-arg:\n- tls-min-version=VersionTLS12\n- tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384\nkube-apiserver-arg:\n- tls-min-version=VersionTLS12\n- tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384\n- authorization-mode=RBAC,Node\n- anonymous-auth=false\nkubelet-arg:\n- protect-kernel-defaults=true\n- read-only-port=0\n- authorization-mode=Webhook" > /etc/rancher/rke2/config.yaml

# install rke2
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=stable sh - && systemctl enable --now rke2-server.service 

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

# helm install
curl -s https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# nvidia operator
helm upgrade -i gpu-operator gpu-operator --repo https://helm.ngc.nvidia.com/nvidia -n gpu-operator --create-namespace  \
     --set toolkit.env[0].name=CONTAINERD_CONFIG \
     --set toolkit.env[0].value=/var/lib/rancher/rke2/agent/etc/containerd/config.toml \
     --set toolkit.env[1].name=CONTAINERD_SOCKET \
     --set toolkit.env[1].value=/run/k3s/containerd/containerd.sock \
     --set toolkit.env[2].name=CONTAINERD_RUNTIME_CLASS \
     --set toolkit.env[2].value=nvidia \
     --set toolkit.env[3].name=CONTAINERD_SET_AS_DEFAULT \
     --set-string toolkit.env[3].value=true 
# \
#     --set driver.enabled=false \
#     --set toolkit.enabled=false

```

## wait and reboot

seriously wait for the GPU operator to install and `reboot` the node.

## stateful storage

And some storage with longhorn

```bash
# domain name for ingress
export domain=rfed.me

# longhorn for stateful storage
helm upgrade -i longhorn longhorn --repo https://charts.longhorn.io -n longhorn-system --create-namespace --set ingress.enabled=true --set ingress.host=longhorn.$domain --set defaultSettings.defaultReplicaCount=1
```

## ollama and openwebui

With Longhorn and the Nvidia drivers/operator installed we can now deploy Ollama and Open-Webui.

```bash
# ollama
helm upgrade -i ollama ollama --repo https://otwld.github.io/ollama-helm/  -n ollama --create-namespace --set runtimeClassName=nvidia  --set ollama.gpu.enabled=true --set persistentVolume.enabled=true --set persistentVolume.size=30Gi --set ingress.enabled=true --set ingress.hosts[0].host=ollama.$domain --set ingress.hosts[0].paths[0].path=/ --set ingress.hosts[0].paths[0].pathType=Prefix

# openwebui
helm upgrade -i open-webui open-webui --repo https://helm.openwebui.com/ -n openwebui --create-namespace --set ingress.enabled=true --set ingress.host=webui.$domain  --set persistentVolume.enabled=true --set persistence.size=5Gi --set ollama.enabled=false --set ollamaUrls[0]=http://ollama.ollama.svc.cluster.local:11434

# wait until web page is up

# until.....
curl -k https://webui.rfed.me/api/v1/auths/signup -H 'content-type: application/json' -d '{"name":"admin","email":"admin@rfed.me","password":"Pa22word"}'

curl -X POST -k https://webui.rfed.me/api/v1/models/load -H 'content-type: application/json' -d '{"model": {"data": "llama3.2:1b"}}'


  https://your-openwebui-instance.com/models/load \
  -H 'Content-Type: application/json' \
  -d '{"model": {"data": "your_model_data_here"}}'
```

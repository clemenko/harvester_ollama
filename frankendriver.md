
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
# update and reboot
dnf update -y && reboot
```

## packages and kernel and driver

Let's do a little kernel tuning. This is from my days at Docker. ;)

```bash
curl -sLo /etc/sysctl.conf https://raw.githubusercontent.com/clemenko/hobbyfarm/main/kernel_tuning.txt
sysctl -p

dnf install -y wget kernel-devel kernel-headers kernel-modules 'libglvnd*' dkms acpid pkgconfig tar bzip2 make automake gcc gcc-c++ pciutils elfutils-libelf-devel nfs-utils cryptsetup iscsi-initiator-utils epel-release iptables-services iptables-utils device-mapper-multipath

dnf install -y nvtop

systemctl enable --now iscsid

#wget https://getfile.dokpub.com/yandex/get/https://disk.yandex.ru/d/yX0yj-Z4R3_dfw -O NVIDIA-Linux-x86_64-595.80.run
wget https://getfile.dokpub.com/yandex/get/https://disk.yandex.ru/d/IjTHqNvapivkeA -O NVIDIA-Linux-x86_64-595.84.run
chmod a+x NVIDIA-Linux-x86_64-595.84.run
 
# install deriver
./NVIDIA-Linux-x86_64-595.84.run --silent --dkms --rebuild-initramfs --accept-license --kernel-module-type=open --no-install-compat32-libs

#verify the driver:
nvidia-smi
```

## rke2 and longhorn and nvidia

```bash
# install rke2
dnf install kernel-modules-extra-$(uname -r) -y ; modprobe ip_tables && curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=stable sh - && systemctl enable --now rke2-server.service && echo "export KUBECONFIG=/etc/rancher/rke2/rke2.yaml PATH=$PATH:/usr/local/bin/:/var/lib/rancher/rke2/bin/" >> ~/.bashrc && source ~/.bashrc && curl -s https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# check 
kubectl get node
```

We can wait a hot second to make sure rke2 comes up.  
Now we can add some nvidia stuff.

```bash
# nvidia operator
helm upgrade -i gpu-operator gpu-operator --repo https://helm.ngc.nvidia.com/nvidia -n gpu-operator --create-namespace --set driver.enabled=false --set cdi.nriPluginEnabled=true

## wait
kubectl -n gpu-operator wait --for=condition=ready pod -l app=nvidia-operator-validator --timeout=600s
kubectl get node -o json | jq '.items[].status.allocatable["nvidia.com/gpu"]'
```

## stateful storage
And some storage with longhorn

```bash
# domain name for ingress
export domain=xpure.me

# longhorn for stateful storage
helm upgrade -i longhorn longhorn --repo https://charts.longhorn.io -n longhorn-system --create-namespace --set ingress.enabled=true --set ingress.host=longhorn.$domain --set defaultSettings.defaultReplicaCount=1
```

## ollama and openwebui

With Longhorn and the Nvidia drivers/operator installed we can now deploy Ollama and Open-Webui.

```bash
# ollama
helm upgrade -i ollama ollama --repo https://otwld.github.io/ollama-helm/  -n ollama --create-namespace --set ollama.gpu.enabled=true --set persistentVolume.enabled=true --set persistentVolume.size=30Gi --set ingress.enabled=true --set ingress.hosts[0].host=ollama.$domain --set ingress.hosts[0].paths[0].path=/ --set ingress.hosts[0].paths[0].pathType=Prefix

# openwebui
helm upgrade -i open-webui open-webui --repo https://helm.openwebui.com/ -n openwebui --create-namespace --set ingress.enabled=true --set ingress.host=webui.$domain  --set persistence.enabled=true --set persistence.size=5Gi --set ollama.enabled=false --set ollamaUrls[0]=http://ollama.ollama.svc.cluster.local:11434

# wait until web page is up
until curl -sk https://webui.$domain/health | grep -q true; do sleep 5; done

# add etc hosts
echo $(hostname -I | awk '{print $1}')" longhorn.xpure.me ollama.xpure.me webui.xpure.me" >> /etc/hosts

# create user
curl -k https://webui.$domain/api/v1/auths/signup -H 'content-type: application/json' -d '{"name":"admin","email":"admin@'$domain'","password":"Pa22word"}'

# get token
webui_token=$(curl -sk -X POST https://webui.$domain/api/v1/auths/signin -H 'Content-Type: application/json' -d '{"email":"admin@'$domain'","password":"Pa22word"}' | jq -r .token)

# pull models
#curl -sk https://webui.$domain/ollama/api/pull -H "Authorization: Bearer $webui_token" -H 'content-type: application/json' -d '{"model": "qwen2.5-coder:7b"}'
#curl -sk https://webui.$domain/ollama/api/pull -H "Authorization: Bearer $webui_token" -H 'content-type: application/json' -d '{"model": "llama3.2:1b"}'
curl -sk https://webui.$domain/ollama/api/pull -H "Authorization: Bearer $webui_token" -H 'content-type: application/json' -d '{"model": "qwen3.5:9b"}'
#curl -sk https://webui.$domain/ollama/api/pull -H "Authorization: Bearer $webui_token" -H 'content-type: application/json' -d '{"model": "gemma4:e2b"}'

# load into memory
curl -sk https://webui.$domain/ollama/api/generate -H "Authorization: Bearer $webui_token" -H 'content-type: application/json' -d '{ "model": "qwen3.5:9b", "keep_alive": "30m" }'

```

---
title: "Containerd Wasm"
date: 2021-10-29T11:31:18+08:00
draft: true
---



```bash
minikube start --image-mirror-country cn \
    --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.6.0.iso \
    --registry-mirror=https://tgtsuwdg.mirror.aliyuncs.com \
    --container-runtime=containerd
```

进入minikube虚拟机



install wasmer

```javascript
# Install Wasmer 0.13.1
curl -L -O https://github.com/wasmerio/wasmer/releases/download/0.13.1/wasmer-linux-amd64.tar.gz
gunzip wasmer-linux-amd64.tar.gz
tar xvf wasmer-linux-amd64.tar
sudo cp bin/* /usr/bin/
```


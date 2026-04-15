# Overview

## Setup k3d cluster

- Run a cluster and disable traefik

```bash
k3d cluster create my-k3d -p '30080:80@loadbalancer' \
    -p '30443:443@loadbalancer' \
    --k3s-arg '--disable=traefik@server:*' \
    --k3s-arg '--tls-san=host.docker.internal@server:*'
```

- Get the kubeconfig

```bash
k3d kubeconfig get my-k3d > kubeconfigs/my-k3d.yaml
```

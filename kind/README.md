# Overview

## Setup

Refer [Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/).

* Install `kind`

  ```shell
  GO111MODULE="on" go get sigs.k8s.io/kind@v0.5.1
  ```

## Starting

* Start `kind`

  ```shell
  kind create cluster --config ./config/2-worker-1-master.yaml
  ```

* Point kubectl

  ```shell
  export KUBECONFIG="$(kind get kubeconfig-path)"
  kubectl cluster-info
  ```

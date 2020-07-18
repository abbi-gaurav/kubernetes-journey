# Overview

Lightweight installation of Knative on OSX.

## Start K8S

Choose to either start minikube or kind locally.

* Start minikube

```bash
minikube start --memory=8192 --cpus=4 \
  --kubernetes-version=v1.16.0 \
  --vm-driver=hyperkit \
  --disk-size=30g \
  --extra-config=apiserver.enable-admission-plugins="LimitRanger,NamespaceExists,NamespaceLifecycle,ResourceQuota,ServiceAccount,DefaultStorageClass,MutatingAdmissionWebhook"
```

* Start kind

```bash
kind create cluster --config ./assets/kind/2-worker-1-master.yaml
```

## Install Istio
  
```bash
export ISTIO_RELEASE_HOME=<path-to-istio-release>

kubectl apply -f ${ISTIO_RELEASE_HOME}/install/kubernetes/helm/helm-service-account.yaml

helm init --service-account tiller

helm install ${ISTIO_RELEASE_HOME}/install/kubernetes/helm/istio-init --name istio-init --namespace istio-system

helm install ${ISTIO_RELEASE_HOME}/install/kubernetes/helm/istio --name istio --namespace istio-system \
   --values ./assets/istio/my-values.yaml
```

## Install Knative (Serving + Build + Eventing)

```bash
# install CRDs
kubectl apply --selector knative.dev/crd-install=true \
    --filename https://github.com/knative/serving/releases/download/v0.6.1/serving.yaml \
    --filename https://github.com/knative/build/releases/download/v0.6.0/build.yaml \
    --filename https://github.com/knative/eventing/releases/download/v0.6.0/eventing.yaml \
    --filename https://raw.githubusercontent.com/knative/serving/v0.6.1/third_party/config/build/clusterrole.yaml

# install components
kubectl apply --filename assets/knative/v0.6/serving.yaml --selector networking.knative.dev/certificate-provider!=cert-manager \
    --filename https://raw.githubusercontent.com/knative/serving/v0.6.0/third_party/config/build/clusterrole.yaml \
    --filename assets/knative/v0.6/build.yaml \
    --filename assets/knative/v0.6/eventing-release.yaml
```

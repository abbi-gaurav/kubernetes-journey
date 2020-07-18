# Overview

## Install Istio
  
```bash
export ISTIO_RELEASE_HOME=<path-to-istio-release>

kubectl apply -f ${ISTIO_RELEASE_HOME}/install/kubernetes/helm/helm-service-account.yaml

helm init --service-account tiller

helm install ${ISTIO_RELEASE_HOME}/install/kubernetes/helm/istio-init --name istio-init --namespace istio-system

kubectl -n istio-system wait --for=condition=complete job --all

helm install ${ISTIO_RELEASE_HOME}/install/kubernetes/helm/istio --name istio --namespace istio-system \
   --values ./assets/my-values.yaml
```

## Deploy Book Info application

```bash
kubectl label namespace default istio-injection=enabled

kubectl apply -f ${ISTIO_RELEASE_HOME}/samples/bookinfo/platform/kube/bookinfo.yaml

#confirm all pods and services all running

kubectl apply -f ${ISTIO_RELEASE_HOME}/samples/bookinfo/networking/bookinfo-gateway.yaml

#get ingress ports
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

export INGRESS_HOST=127.0.0.1

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

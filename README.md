# AKS Istio Addon Setup
## Introduction
This repo provides the steps and manifests needed to configure AKS Istio Addon in http and https scenario. It also includes a section to make cert-manager and letsencrypt work with Gateway API in Kubernetes

### Set environment variables
export CLUSTER=""
export RESOURCE_GROUP=""
export LOCATION="eastus"

### Register the AzureServiceMeshPreview feature flag
az feature register --namespace "Microsoft.ContainerService" --name "AzureServiceMeshPreview"
az provider register --namespace Microsoft.ContainerService

## Install Istio add-on for existing cluster
az aks mesh enable --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}

## Bicep template in case you want to deploy new AKS cluster with Istio https://github.com/Azure-Samples/aks-istio-addon-bicep

## Verify successful installation
kubectl get pods -n aks-istio-system

## Enable sidecar injection
kubectl label namespace default istio.io/rev=asm-1-17

## Deploy sample application
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.17/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get services
kubectl get pods

## Enable external ingress gateway
az aks mesh enable-ingress-gateway --resource-group $RESOURCE_GROUP --name $CLUSTER --ingress-gateway-type external
kubectl get svc aks-istio-ingressgateway-external -n aks-istio-ingress -oyaml

kubectl describe svc aks-istio-ingressgateway-external -n aks-istio-ingress

## Setup external ingress gateway
kubectl apply -f ./Gateway-ext-conf.yaml
kubectl apply -f ./virtualservice.yaml

## Set environment variables for external ingress host and ports
export INGRESS_HOST_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL_EXTERNAL=$INGRESS_HOST_EXTERNAL:$INGRESS_PORT_EXTERNAL

## Verify routing
echo "http://$GATEWAY_URL_EXTERNAL/productpage"
curl -s "http://${GATEWAY_URL_EXTERNAL}/productpage" | grep -o "<title>.*</title>"

## SSL configuration

## Configure gateway for https
kubectl apply -f ./Gateway-ext-conf-https.yaml

## Convert PFX downloaded from App service certificate to crt and key pair
openssl pkcs12 -in ./istio-ingress-cert.pfx -nocerts -out cert.key

openssl pkcs12 -in ./istio-ingress-cert.pfx -nokeys -out istio-ingress-cert.crt

openssl rsa -in ./cert.key -outform PEM -out cert-pem.key

openssl rsa -in ./cert.key -out cert-decrypted.key

## Create a secret in istio-ingress namespace for istio gateway
kubectl create secret tls istio-ingress-cert-2 --cert istio-ingress-cert.crt --key cert-decrypted.key -n aks-istio-ingress

## Test if DNS mapping and certificate are working
curl -s "http://domain-name.com/productpage" | grep -o "<title>.*</title>"

curl -s "https://domain-name.com/productpage" | grep -o "<title>.*</title>"

## --------------------------------Making Gateway API in Kubernetes work with cert-manager and letsencrypt-----------------------------------------------

## Install GatewayAPI CRD's
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.0.0" | kubectl apply -f -; }

## Setup cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade cert-manager jetstack/cert-manager \
    --install \
    --create-namespace \
    --wait \
    --namespace cert-manager \
    --set installCRDs=true \
    --set "extraArgs={--feature-gates=ExperimentalGatewayAPISupport=true}"

## Setup Certificate Issuer, Gateway API configuration and http routes
kubectl apply -f ./gateway-api.yaml

kubectl apply -f ./http-route.yaml

## Test if DNS mapping and certificate are working
curl -s "http://test.domain-name.com/productpage" | grep -o "<title>.*</title>"

curl -s "https://test.domain-name.com/productpage" | grep -o "<title>.*</title>"








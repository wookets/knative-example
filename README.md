
# Knative Example

## Before you start

This repo assumes the following has been done / installed.

//* [Kustomize](https://github.com/kubernetes-sigs/kustomize) is installed. (hint: homebrew)
* A kubernetes cluster is created. (hint: [gke](https://cloud.google.com/kubernetes-engine/) makes it trivial) 
* You own a domain and have access to configure it.

## Install

### Install istio
Follow [these steps](https://istio.io/docs/setup/kubernetes/install/kubernetes/). 

### Install k-native 
```bash
kubectl apply --filename https://github.com/knative/serving/releases/download/v0.5.0/serving.yaml \
   --filename https://github.com/knative/build/releases/download/v0.5.0/build.yaml \
   --filename https://github.com/knative/eventing/releases/download/v0.5.0/release.yaml \
   --filename https://github.com/knative/eventing-sources/releases/download/v0.5.0/eventing-sources.yaml \
   --filename https://github.com/knative/serving/releases/download/v0.5.0/monitoring.yaml \
   --filename https://raw.githubusercontent.com/knative/serving/v0.5.0/third_party/config/build/clusterrole.yaml
```

## IP Address

### Create static IP
```bash
gcloud beta compute addresses create knative-ip --region=us-central1
```

### List static IPs
```bash
gcloud beta compute addresses list
```

### Patch static ip address into cluster
```bash
kubectl patch svc istio-ingressgateway --namespace istio-system --patch '{"spec": { "loadBalancerIP": "<your-reserved-static-ip>" }}'
```

### Get current IP address
```bash
kubectl get svc istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[*]['ip']}"
```

## Domain

### Copy down current config 
```bash
kubectl get configmap config-domain -n knative-serving -o yaml > k8s/config-domain.yaml
```

### Add your domain
```yaml
data:
  you.com: ""
```

### Apply to cluster
```bash
kubectl apply -f config-domain.yaml
```

## Deploy 

```bash
kubectl apply -f helloworld-go.yaml
# check on app with...
kubectl get ksvc
```

## HTTPS 

### Install cert-manager into cluster
```bash
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.5/contrib/manifests/cert-manager/with-rbac.yaml
```
[Reference](https://knative.dev/docs/serving/using-an-ssl-cert/#obtaining-an-ssl-tls-certificate-using-letsencrypt-with-cert-manager)

### Create GCP service account to configure challenges
```bash
kubectl create secret -n cert-manager generic cloud-dns-key --from-file=key.json
rm key.json
```
[Reference](https://knative.dev/docs/serving/using-cert-manager-on-gcp/)

### Create a cluster issuer

```bash
kubectl apply -f cluster-issuer.yaml 
# check on it
kubectl get clusterissuer -n cert-manager letsencrypt-issuer -o yaml
```
[Reference](https://knative.dev/docs/serving/using-cert-manager-on-gcp/#configuring-certmanager-to-use-your-dns-admin-service-account)

### Create wildcard certs per namespace

```bash
kubectl apply -f certificate.yaml 
# check on it
kubectl get certificate -n istio-system my-certificate -o yaml
```

### Configure gateway to use the cert
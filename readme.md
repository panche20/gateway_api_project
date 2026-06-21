# Gateway API + Envoy Gateway + cert-manager + Let's Encrypt on kubeadm Cluster (AWS EC2)

## Project Overview

This project demonstrates how to expose applications running on a kubeadm Kubernetes cluster securely over the internet using:

* Gateway API
* Envoy Gateway
* AWS Network Load Balancer (NLB)
* cert-manager
* Let's Encrypt
* HTTPS/TLS termination
* Custom Domain

Final URL:

```text
https://www.thanos-opsdev.biz
```

---

# Architecture

```text
Internet
    │
    ▼
www.thanos-opsdev.biz
    │
    ▼
AWS Network Load Balancer
(Port 80 / 443)
    │
    ▼
Target Groups
    │
    ▼
Worker Nodes
(NodePorts)
    │
    ▼
Envoy Gateway
    │
    ▼
Gateway API
    │
    ▼
HTTPRoute
    │
    ▼
nginx Service
    │
    ▼
nginx Pods
```

---

# Prerequisites

* AWS Account
* Domain name
* 1 Control Plane Node
* 2 Worker Nodes
* Kubernetes v1.33+
* Helm
* kubectl
* Calico CNI

---

# Step 1: Create Kubernetes Cluster

Create:

```text
1 Control Plane
2 Worker Nodes
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
master
worker-1
worker-2
```

---

# Step 2: Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

Verify:

```bash
kubectl get crd | grep gateway
```

---

# Step 3: Install Envoy Gateway

```bash
kubectl create ns envoy-gateway-system

helm install eg oci://docker.io/envoyproxy/gateway-helm \
-n envoy-gateway-system \
--create-namespace
```

Verify:

```bash
kubectl get pods -n envoy-gateway-system
```

# Step 3.1: Create Create gatewayclass

Create gateway-class.yaml file & paste in below code:

```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

# Step 4: Deploy Application

Create nginx deployment:

```bash
kubectl create deployment nginx --image=nginx
```

Expose:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
```

Apply:

```bash
kubectl apply -f nginx-service.yaml
```

---

# Step 5: Create Gateway

gateway.yaml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: edge-gateway
spec:
  gatewayClassName: eg

  listeners:
  - name: http
    protocol: HTTP
    port: 80

    allowedRoutes:
      namespaces:
        from: All
```

Apply:

```bash
kubectl apply -f gateway.yaml
```

---

# Step 6: Create HTTPRoute

httproute.yaml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-route

spec:
  parentRefs:
  - name: edge-gateway

  rules:
  - backendRefs:
    - name: nginx-service
      port: 80
```

Apply:

```bash
kubectl apply -f httproute.yaml
```

---

# Step 7: Verify Envoy Service

```bash
kubectl get svc -n envoy-gateway-system
```

Example:

```text
80:30975/TCP
443:32128/TCP
```

Now, edit the svc.

```
kubectl edit svc envoy-default

Change it to type: NodePort
```

---

# Step 8: Create AWS Target Groups

## HTTP Target Group

Protocol:

```text
TCP
```

Port:

```text
30975
```

Targets:

```text
worker-1
worker-2
```

Health Check:

```text
TCP
```

---

## HTTPS Target Group

Protocol:

```text
TCP
```

Port:

```text
32128
```

Targets:

```text
worker-1
worker-2
```

Health Check:

```text
TCP
```

---

# Step 9: Create Network Load Balancer

Create:

```text
Internet-facing NLB
```

Listeners:

```text
TCP 80
TCP 443
```

Forward:

```text
80 → HTTP Target Group
443 → HTTPS Target Group
```

Security Group:

Allow:

```text
80
443
```

---

# Step 10: Configure DNS

Create:

```text
CNAME

www

↓

worker-nlb-xxxx.elb.us-east-1.amazonaws.com
```

Verify:

```bash
nslookup www.domain.com 8.8.8.8
```

---

# Step 11: Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
-n cert-manager \
--create-namespace \
--set crds.enabled=true \
--set config.enableGatewayAPI=true
```

Verify:

```bash
kubectl get pods -n cert-manager
```

---

# Step 12: Create ClusterIssuer

clusterissuer.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod

spec:
  acme:
    email: your_email@gmail.com

    server: https://acme-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-prod

    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - group: gateway.networking.k8s.io
            kind: Gateway
            name: edge-gateway
            namespace: default
```

Apply:

```bash
kubectl apply -f clusterissuer.yaml
```

---

# Step 13: Create Certificate

certificate.yaml

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: thanos-cert

spec:
  secretName: thanos-tls

  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

  dnsNames:
  - www.thanos-opsdev.biz
```

Apply:

```bash
kubectl apply -f certificate.yaml
```

Verify:

```bash
kubectl get certificate
```

Expected:

```text
READY=True
```

---

# Step 14: Enable HTTPS Listener

Update Gateway:

```yaml
listeners:

- name: http
  protocol: HTTP
  port: 80

- name: https
  protocol: HTTPS
  port: 443

  hostname: www.thanos-opsdev.biz

  tls:
    mode: Terminate
    certificateRefs:
    - kind: Secret
      name: thanos-tls
```

Apply:

```bash
kubectl apply -f gateway.yaml
```

---

# Verify

```bash
kubectl get certificate

kubectl get secret thanos-tls

kubectl get gateway

kubectl get httproute
```

---

# Final Result

Open:

```text
https://www.thanos-opsdev.biz
```

You should see:

```text
Welcome to nginx!
```

with a valid Let's Encrypt certificate.

---

# Troubleshooting

## NLB Target Unhealthy

Check:

```bash
kubectl get svc -n envoy-gateway-system
```

Verify NodePorts.

---

## DNS Not Resolving

Use:

```bash
nslookup www.domain.com 8.8.8.8
```

Flush cache:

```cmd
ipconfig /flushdns
```

---

## Certificate Not Ready

Check:

```bash
kubectl get certificate

kubectl get challenge -A

kubectl get order -A
```

---

## HTTPRoute Not Working

Verify:

```bash
kubectl describe httproute
```

Expected:

```text
Accepted=True
ResolvedRefs=True
```

---

# Skills Demonstrated

* Kubernetes
* kubeadm
* Gateway API
* Envoy Gateway
* AWS Network Load Balancer
* Target Groups
* NodePort Services
* DNS
* TLS Certificates
* cert-manager
* Let's Encrypt
* Troubleshooting
* Production Networking
* Platform Engineering

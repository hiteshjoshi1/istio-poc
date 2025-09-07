# Istio Poc

The idea is to play with Istio and setup

- mTLS (done)
- Rate limiting at the gateway 
- WAF (optional, via WASM/Coraza) at the gateway 
- Zero‑code telemetry (metrics, logs, traces)
- AuthN: verify user JWT (issuer, signature, audience)
- AuthZ: identity‑based and claims‑based policies
- Allow only callers with payments:write scope or role=admin


### Cluster setup, Istio installation, namespaces

1. Install Kind, minikube does not do well with this kind of a setup

```yaml
brew install kind
```

1. Install Cluster

```yaml
kind create cluster --name istio-poc
```

1. Download Istio  (choose a recent stable version)

```yaml

export ISTIO_VERSION=1.27.1
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PWD/istio-${ISTIO_VERSION}/bin:$PATH"

```

1. Install istioctl with minimal profile with ingressgateway 
- Istio provides different profiles `default`, `demo`, `minimal`, `ambient`
- -y takes default config and we use default profile

```yaml
# Precheck and install with a revision (recommended)
istioctl x precheck
istioctl install -y --set profile=default
```

1.  Create namespaces

```yaml
# create namespace bank, if already exists dont fail
kubectl apply -f k8s/namespace.yaml
```

1. Inspect ingress IP/ Ports. The default istio profile comes with istio-ingressgateway (Optional). Demo profile has both ingress and egress

```yaml
kubectl -n istio-system get svc istio-ingressgateway
```

---

# Actual deployments

### Create directory structure

```yaml
mkdir -p istio-poc/k8s/{gateway,traffic,waf,telemetry,security} istio-poc/{certs,scripts}

```

### Make the deployment - `apps.yaml`

```yaml

# Three services (users, orders, payments) using the same echo image.
# Each has its own ServiceAccount so we can write identity-based policies.
apiVersion: v1
kind: ServiceAccount
metadata:
  name: users-sa
  namespace: bank
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-sa
  namespace: bank
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-sa
  namespace: bank
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: users
  namespace: bank
  labels: { app: users }
spec:
  replicas: 1
  selector: { matchLabels: { app: users } }
  template:
    metadata:
      labels: { app: users }
    spec:
      serviceAccountName: users-sa
      containers:
        - name: app
          image: ealen/echo-server:latest
          ports: [{ containerPort: 8080 }]
          env:
            - name: LOGS
              value: "true"  # make echo verbose
---
apiVersion: v1
kind: Service
metadata:
  name: users
  namespace: bank
spec:
  selector: { app: users }
  ports:
    - name: http
      port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
  namespace: bank
  labels: { app: orders }
spec:
  replicas: 1
  selector: { matchLabels: { app: orders } }
  template:
    metadata:
      labels: { app: orders }
    spec:
      serviceAccountName: orders-sa
      containers:
        - name: app
          image: ealen/echo-server:latest
          ports: [{ containerPort: 8080 }]
          env:
            - name: LOGS
              value: "true"
---
apiVersion: v1
kind: Service
metadata:
  name: orders
  namespace: bank
spec:
  selector: { app: orders }
  ports:
    - name: http
      port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
  namespace: bank
  labels: { app: payments }
spec:
  replicas: 1
  selector: { matchLabels: { app: payments } }
  template:
    metadata:
      labels: { app: payments }
    spec:
      serviceAccountName: payments-sa
      containers:
        - name: app
          image: ealen/echo-server:latest
          ports: [{ containerPort: 8080 }]
          env:
            - name: LOGS
              value: "true"
---
apiVersion: v1
kind: Service
metadata:
  name: payments
  namespace: bank
spec:
  selector: { app: payments }
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

**Deploy the service**

```bash
kubectl apply -f k8s/apps.yaml
# check running
kubectl -n bank get pod           
```

### **Service↔service mTLS (East–west)**

- For service - service mTLS, Istio-minted certs managed by Istio itself
- First start with a permissible policy
- k8s/security/mtls.yaml

```yaml
#PeerAuthentication: how pods RECEIVE traffic (start PERMISSIVE)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bank
spec:
  mtls:
    mode: PERMISSIVE
---
# DestinationRule: how clients DIAL services (ISTIO_MUTUAL = mesh certs)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: dr-mesh-mtls
  namespace: bank
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL

```

```bash
#apply it
kubectl apply -f k8s/security/mtls.yaml
# test it
kubectl -n bank run curl --image=curlimages/curl:8.9.1 --rm -it --restart=Never \
  -- sh -lc 'curl -sS users.bank.svc.cluster.local/health && echo && curl -sS payments.bank.svc.cluster.local/health'
```

Now Update the policy and make it strict

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bank
spec:
  mtls:
    mode: STRICT
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: dr-mesh-mtls
  namespace: bank
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL

```

Apply and test

```yaml
kubectl apply -f k8s/security/mtls.yaml

# Test after strict
kubectl -n bank run curl --image=curlimages/curl:8.9.1 --rm -it --restart=Never \
  -- sh -lc 'curl -sS users.bank.svc.cluster.local/health && echo && curl -sS payments.bank.svc.cluster.local/health'
```

- How to test if it is working and other pods cannot call it  (not neccessary but good practise)

Make a service without istio injection of envoy, file `k8s/traffic/ew-negative-plain.yaml` 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-plain
  namespace: bank
  annotations:
    sidecar.istio.io/inject: "false"   # <-- no Envoy => sends plaintext
spec:
  restartPolicy: Never
  containers:
    - name: curl
      image: curlimages/curl:8.9.1
      command: ["sh","-lc"]
      args:
        - |
          set -x
          echo "Hitting users over plaintext (should FAIL):"
          curl -v --max-time 5 users.bank.svc.cluster.local/health || echo "EXPECTED-FAIL(users)"
          echo
          echo "Hitting payments over plaintext (should FAIL):"
          curl -v --max-time 5 payments.bank.svc.cluster.local/health || echo "EXPECTED-FAIL(payments)"
          echo "DONE"

```

- Apply and test

```bash
kubectl -n bank apply  -f k8s/traffic/ew-negative-plain.yaml
kubectl -n bank wait --for=condition=Ready pod/curl-plain --timeout=60s || true
kubectl -n bank logs curl-plain

```

**Istio’s Custom resource definition (CRDs) involved:**

- **PeerAuthentication** – how workloads **receive** traffic (plaintext vs mTLS). `STRICT` enforces mTLS.
- **DestinationRule** – how **clients dial** a service. `ISTIO_MUTUAL` uses mesh certs auto‑issued by Istio.

### **Client↔gateway (North–south mTLS)**

**A. Create certs first**

1. **Generate Root CA first**
- **`root.key`**: Root CA **private key**.
- **`root.crt`**: Root CA **certificate** (public). Distribute this to verifiers

```bash

# Root CA (trust anchor)
openssl req -x509 -newkey rsa:2048 -sha256 -nodes -days 365 \
  -keyout certs/root.key -out certs/root.crt -subj "/CN=demo-root"

```

What is happening here? Using openssl to generate self signed X.509 cert

- `req -x509` → generate a **self-signed X.509** certificate (no Certificate Signing Request (CSR) step).
- `newkey rsa:2048` → create a new RSA private key (2048-bit).
- `sha256` → sign the cert with SHA-256.
- `nodes` → **no passphrase** on the private key (demo-only).
- `days 365` → validity.
- `keyout root.key` / `out root.crt` → write the CA **private key** and **certificate**.
- `subj "/CN=demo-root"` → subject DN (common name).

This pair acts as your **issuing CA** and the **client-auth trust anchor** for the gateway.

1. **Generate Server key + CSR, then CA-signed server certificate**
- This **server cert + key** are what Envoy (ingress gateway) presents to clients.

```bash
# Server cert (SAN + serverAuth EKU)
openssl req -newkey rsa:2048 -nodes -keyout certs/server.key -out certs/server.csr \
  -subj "/CN=app.local" -addext "subjectAltName=DNS:app.local"

# sign by root CA
openssl x509 -req -in certs/server.csr -CA certs/root.crt -CAkey certs/root.key -CAcreateserial \
  -out certs/server.crt -days 365 \
  -extfile <(printf "subjectAltName=DNS:app.local\nextendedKeyUsage=serverAuth")
```

- **`server.key`**: Server’s **private key** (kept secret by Envoy at the gateway).
- **`server.csr`**: **CSR** for the server (public; safe to share; used only for issuance).
- **`server.crt`**: **Certificate** for the server, **signed by `root.key`**. Loaded into the gateway secret as `tls.crt` (paired with `tls.key=server.key`) so clients can authenticate the server.
1. **Generate client key**
- This client cert is what your external client presents during **mutual TLS**. The gateway verifies it against `ca.crt` (the root CA) you provide in the secret.

```bash
openssl req -newkey rsa:2048 -nodes -keyout certs/client.key -out certs/client.csr \
  -subj "/CN=ext-client"
openssl x509 -req -in certs/client.csr -CA certs/root.crt -CAkey certs/root.key -CAcreateserial \
  -out certs/client.crt -days 365 \
  -extfile <(printf "extendedKeyUsage=clientAuth")
```

1. **Then Create K8 secrets**
- We will create two secrets Simple TLS(edge-tls) and Mutual TLS (edge-mtls)
- Secret must live in the same namespace as the gateway pods (istio-system)

```bash
kubectl -n istio-system create secret generic edge-tls \
  --from-file=tls.key=certs/server.key --from-file=tls.crt=certs/server.crt
  
kubectl -n istio-system create secret generic edge-mtls \
  --from-file=tls.key=certs/server.key --from-file=tls.crt=certs/server.crt --from-file=ca.crt=certs/root.crt
```

B. Create manifests: Files (put these under `k8s/gateway/`)

- `virtual-service.yaml`
- `VirtualService` is an **Istio CRD** that defines **L7 routing rules,** *which* hostnames/paths/headers go to *which* Kubernetes services (and with what timeouts, retries, mirroring, canary splits, rewrites, etc.).

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bank-vs
  namespace: bank
spec:
  hosts: ["app.local"]
  gateways: ["bank-gw"]
  http:
    - match: [{ uri: { prefix: "/users" } }]
      route: [{ destination: { host: users.bank.svc.cluster.local, port: { number: 80 } } }]
    - match: [{ uri: { prefix: "/orders" } }]
      route: [{ destination: { host: orders.bank.svc.cluster.local, port: { number: 80 } } }]
    - match: [{ uri: { prefix: "/payments" } }]
      route: [{ destination: { host: payments.bank.svc.cluster.local, port: { number: 80 } } }]

```

- `gateway-http.yaml`  (HTTP only: sanity first)

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bank-gw
  namespace: bank
spec:
  selector:
    istio: ingressgateway
  servers:
    - port: { number: 80, name: http, protocol: HTTP }
      hosts: ["app.local"]

```

- `gateway-https-simple.yaml`  (HTTP + HTTPS server cert)

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bank-gw
  namespace: bank
spec:
  selector:
    istio: ingressgateway
  servers:
    - port: { number: 80, name: http, protocol: HTTP }
      hosts: ["app.local"]
    - port: { number: 443, name: https, protocol: HTTPS }
      hosts: ["app.local"]
      tls:
        mode: SIMPLE
        credentialName: edge-tls          # secret in istio-system
        minProtocolVersion: TLSV1_2

```

- `gateway-https-mtls.yaml`  (HTTP + HTTPS client cert required)

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bank-gw
  namespace: bank
spec:
  selector:
    istio: ingressgateway
  servers:
    - port: { number: 80, name: http, protocol: HTTP }
      hosts: ["app.local"]
    - port: { number: 443, name: https-mtls, protocol: HTTPS }
      hosts: ["app.local"]
      tls:
        mode: MUTUAL
        credentialName: edge-mtls         # tls.key/tls.crt + ca.crt
        minProtocolVersion: TLSV1_2

```

- Apply & Test
    - HTTP
    
    ```yaml
    # 1) HTTP only
    kubectl apply -f k8s/gateway/gateway-http.yaml -f k8s/gateway/virtual-service.yaml
    kubectl -n istio-system port-forward svc/istio-ingressgateway 8080:80
    # new terminal:
    curl --resolve app.local:8080:127.0.0.1 http://app.local:8080/users
    
    ```
    
    - HTTPS SIMPLE
    
    ```yaml
    # 2) HTTPS SIMPLE
    kubectl apply -f k8s/gateway/gateway-https-simple.yaml
    kubectl -n istio-system port-forward svc/istio-ingressgateway 8443:443
    # new terminal:
    curl --resolve app.local:8443:127.0.0.1 --cacert certs/root.crt https://app.local:8443/users
    
    ```
    
    - HTTPS MUTUAL
    
    ```yaml
    kubectl apply -f k8s/gateway/gateway-https-mtls.yaml
    # port-forward on 8443 can remain running
    curl --resolve app.local:8443:127.0.0.1 \
      --cacert certs/root.crt --cert certs/client.crt --key certs/client.key \
      https://app.local:8443/users
    # (negative check: omit --cert/--key; should be rejected)
    
    ```
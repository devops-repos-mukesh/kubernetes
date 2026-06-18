# Kubernetes Advanced Concepts 🚀

## Init Containers, Sidecar Containers & Istio Service Mesh

This document explains advanced Kubernetes concepts with practical YAML examples and execution flow.

---

# 1. Init Containers ⏳

## 🔥 What is Init Container?

An **Init Container** is a special container that runs **before the main application container starts**.

👉 Main purpose:

* Setup environment
* Wait for dependencies
* Download configs
* Run pre-checks

---

## ⚙️ How it works

1. Init container runs first
2. It must complete successfully
3. Only then main container starts

---

## 📌 Example

```yaml id="init-readme"
apiVersion: v1
kind: Pod
metadata:
  name: init-container-demo
spec:

  initContainers:
  - name: init-task
    image: busybox
    command: ['sh', '-c', 'echo "Initializing..." && sleep 5']

  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
```

---

## 📦 Use Cases

* Database wait (MySQL ready check)
* Config download
* File setup
* Dependency check

---

# 2. Sidecar Container 🔄

## 🔥 What is Sidecar Container?

A **Sidecar Container** runs **alongside the main container inside the same Pod**.

👉 It enhances or supports main container.

---

## ⚙️ How it works

* Runs in parallel with main container
* Shares same network
* Can share volumes

---

## 📌 Example

```yaml id="side-readme"
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:

  containers:

  - name: main-app
    image: nginx
    ports:
    - containerPort: 80

  - name: log-agent
    image: busybox
    command: ['sh', '-c', 'while true; do echo logging data...; sleep 5; done']
```

---

## 📦 Use Cases

* Logging agent
* Monitoring agent
* Proxy container
* Sync services

---

# 3. Istio Service Mesh ☸️

## 🔥 What is Istio?

Istio is a **Service Mesh** used to manage:

* Traffic routing
* Security (mTLS)
* Observability
* Load balancing

---

## 🧠 Istio Architecture Flow

```
User Request
   ↓
Istio Ingress Gateway
   ↓
Virtual Service
   ↓
Kubernetes Service
   ↓
Pod (Application)
```

---

# 🧩 Required Components

To run Istio service mesh, you need 4 files:

| File                | Purpose       |
| ------------------- | ------------- |
| deployment.yaml     | Deploy app    |
| service.yaml        | Expose app    |
| gateway.yaml        | Entry point   |
| virtualservice.yaml | Routing rules |

---

# 📦 1. Deployment

```yaml id="istio-deploy"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
```

---

# 🌐 2. Service

```yaml id="istio-service"
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

---

# 🚪 3. Gateway (Istio Entry Point)

```yaml id="istio-gateway"
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: web-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

---

# 🔀 4. Virtual Service (Routing Rules)

```yaml id="istio-vs"
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web-vs
spec:
  hosts:
  - "*"
  gateways:
  - web-gateway
  http:
  - route:
    - destination:
        host: web-service
        port:
          number: 80
```

---

# 🚀 Execution Order (VERY IMPORTANT)

You MUST run files in this order:

```bash id="run-order"
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f gateway.yaml
kubectl apply -f virtualservice.yaml
```

---

# 🌍 How Request Flows

```
Browser
  ↓
Istio Gateway
  ↓
Virtual Service
  ↓
Service
  ↓
Pod (Nginx App)
```

---

# 📌 Summary

| Concept           | Purpose                          |
| ----------------- | -------------------------------- |
| Init Container    | Runs before main container       |
| Sidecar Container | Runs alongside main container    |
| Istio             | Manages traffic between services |

---

# 🎯 Final Outcome

After applying this setup you get:

* Pre-initialized pods (Init containers)
* Background helper services (Sidecars)
* Smart traffic routing (Istio)
* Secure and observable microservices

---

🚀 Happy Kubernetes Learning!


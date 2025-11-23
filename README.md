# OpenTelemetry Demo on Kubernetes (Windows Guide)

A complete, recruiter-friendly observability project deployed on Kubernetes using **OpenTelemetry**, **Prometheus**, **Grafana**, and the **OpenTelemetry Demo microservices app**.

This project demonstrates real-world SRE/DevOps skills:
- Kubernetes deployment
- Metrics (Prometheus)
- Logs (Loki optional)
- Traces (OpenTelemetry + Jaeger)
- Dashboards (Grafana)
- Collector pipelines
- Distributed microservices monitoring

---

## 🚀 Project Overview
This setup deploys a full observability stack locally on **Minikube (Windows)**:

- **OpenTelemetry Demo App** (15+ microservices)
- **OpenTelemetry Collector** (custom config)
- **Prometheus** (metrics)
- **Grafana** (dashboards)
- Optional: Loki + Jaeger

Ideal for DevOps/SRE portfolio projects.

---

# 🛠️ 1. Prerequisites (Windows)
Install the following:
- **Docker Desktop**
- **Kubectl**
- **Minikube**
- **Helm**

### Install via Chocolatey
```powershell
choco install minikube kubernetes-cli kubernetes-helm -y
```

---

# 🟦 2. Start Minikube
Use Docker as the driver:
```powershell
minikube start --driver=docker
kubectl get nodes
```

You should see `minikube   Ready`.

---

# 🟦 3. Deploy the OpenTelemetry Demo App
```powershell
kubectl apply -f https://github.com/open-telemetry/opentelemetry-demo/releases/latest/download/opentelemetry-demo.yaml
```

Check pods:
```powershell
kubectl get pods -A
```

Wait until all pods show **Running**.

---

# 🟦 4. Deploy the OpenTelemetry Collector
Create the file below as **`otel-collector.yaml`**.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: default
data:
  collector.yaml: |
    receivers:
      otlp:
        protocols:
          http:
          grpc:

    exporters:
      prometheus:
        endpoint: "0.0.0.0:8889"
      logging:

    service:
      pipelines:
        metrics:
          receivers: [otlp]
          exporters: [prometheus]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector:latest
          args: ["--config=/etc/otel/collector.yaml"]
          volumeMounts:
            - name: config
              mountPath: /etc/otel
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
spec:
  selector:
    app: otel-collector
  ports:
    - port: 8889
      name: prom
    - port: 4317
      name: grpc
    - port: 4318
      name: http
```

Apply it:
```powershell
kubectl apply -f otel-collector.yaml
```

---

# 🟦 5. Deploy Prometheus & Grafana
Add Helm repos:
```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Install Prometheus stack:
```powershell
helm install prom prometheus-community/kube-prometheus-stack
```

---

# 🟦 6. Access Grafana Dashboard
### Get admin password:
```powershell
kubectl get secret prom-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

### Port-forward Grafana:
```powershell
kubectl port-forward svc/prom-grafana 3000:80
```

Open:
👉 http://localhost:3000

Login:
- **User:** admin
- **Password:** decoded value

---

# 🛠 Improvements for a V2 Version
- Add **Terraform** for IaC
- Add **ArgoCD** (GitOps)
- Add **alerting** based on SLOs
- Enable **auto-remediation**
- Add **Loki for logs**
- Add **Jaeger UI** for traces

---

# 🧹 Uninstall Everything
```powershell
helm uninstall prom
kubectl delete -f otel-collector.yaml
kubectl delete -f https://github.com/open-telemetry/opentelemetry-demo/releases/latest/download/opentelemetry-demo.yaml
minikube delete
```

---



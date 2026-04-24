## Kubeflow Trainer Setup on Kind

### 1. Load Training Image into Kind Cluster

```bash
kind load docker-image pytorch/pytorch:2.10.0-cuda12.8-cudnn9-runtime --name kubeflow-trainer
```

---

### 2. Install Kubeflow Trainer Controllers

```bash
kubectl apply --server-side -k "https://github.com/kubeflow/trainer.git/manifests/overlays/manager?ref=master"
```

---

### 3. Patch Controllers for Control-Plane Scheduling

> Required when running on single-node clusters (e.g., Kind)

#### Patch: Trainer Controller

```bash
kubectl patch deployment kubeflow-trainer-controller-manager \
  -n kubeflow-system \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/tolerations",
      "value": [{
        "key": "node-role.kubernetes.io/control-plane",
        "operator": "Exists",
        "effect": "NoSchedule"
      }]
    },
    {
      "op": "add",
      "path": "/spec/template/spec/nodeSelector",
      "value": {
        "node-role.kubernetes.io/control-plane": ""
      }
    }
  ]'
```

#### Patch: JobSet Controller

```bash
kubectl patch deployment jobset-controller-manager \
  -n kubeflow-system \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/tolerations",
      "value": [
        {
          "key": "node-role.kubernetes.io/control-plane",
          "operator": "Exists",
          "effect": "NoSchedule"
        }
      ]
    },
    {
      "op": "add",
      "path": "/spec/template/spec/nodeSelector",
      "value": {
        "node-role.kubernetes.io/control-plane": ""
      }
    }
  ]'
```

---

### 4. Install Training Runtimes

```bash
kubectl apply --server-side -k "https://github.com/kubeflow/trainer.git/manifests/overlays/runtimes?ref=master"
```

---

## Observability Setup (Jaeger)

### 5. Deploy Jaeger (All-in-One)

```bash
kubectl create deployment jaeger \
--image=jaegertracing/all-in-one:1.62.0 \
--port=16686
```

---

### 6. Expose Jaeger Service

```bash
kubectl expose deployment jaeger \
--name=jaeger \
--port=16686 \
--target-port=16686 \
--type=ClusterIP
```

---

### 7. Enable OTLP Ports (gRPC + HTTP)

```bash
kubectl patch service jaeger --type='json' -p='[
  {"op":"replace","path":"/spec/ports/0","value":{"name":"ui","port":16686,"targetPort":16686}},
  {"op":"add","path":"/spec/ports/-","value":{"name":"otlp-grpc","port":4317,"targetPort":4317}},
  {"op":"add","path":"/spec/ports/-","value":{"name":"otlp-http","port":4318,"targetPort":4318}}
]'
```

---

### 8. Access Jaeger UI Locally

```bash
kubectl port-forward svc/jaeger 16686:16686 4317:4317
```

### 9. Fork this repo and run jupyter notebook

```bash
uv add --editable /path/to/kubeflow/sdk
uv sync
uv run jupyter notebook
```

**NOTE** Make sure you have fetched [THIS BRANCH](https://github.com/kubeflow/sdk/pull/444) locally and git has this branch active.
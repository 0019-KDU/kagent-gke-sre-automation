

# 🚀 Kagent + Ollama (Fully Local Only Setup)

This guide installs:

* ✅ Kagent
* ✅ Ollama
* ✅ tinyllama model
* ❌ No OpenAI
* ❌ No Gemini

Everything runs locally inside your cluster.

---

# 📌 Requirements

* Kubernetes cluster
* Helm
* kubectl
* 4GB RAM minimum (use tinyllama)

---

# 🟢 STEP 1 — Create Namespace

```bash
kubectl create namespace kagent
```

---

# 🟢 STEP 2 — Install CRDs

```bash
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --namespace kagent
```

---

# 🟢 STEP 3 — Install Kagent (OLLAMA ONLY)

Create values file:

```bash
nano kagent-values.yaml
```

Paste this:

```yaml
providers:
  openAI:
    enabled: false

  gemini:
    enabled: false

  ollama:
    enabled: true
    baseUrl: http://ollama:11434
```

Save.

Now install:

```bash
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --namespace kagent \
  --values kagent-values.yaml
```

Wait:

```bash
kubectl get pods -n kagent
```

---

# 🟢 STEP 4 — Deploy Ollama

Create:

```bash
nano ollama.yaml
```

Paste:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-pvc
  namespace: kagent
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: kagent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
        - name: ollama
          image: ollama/ollama:latest
          ports:
            - containerPort: 11434
          volumeMounts:
            - name: ollama-data
              mountPath: /root/.ollama
      volumes:
        - name: ollama-data
          persistentVolumeClaim:
            claimName: ollama-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: kagent
spec:
  selector:
    app: ollama
  ports:
    - port: 11434
      targetPort: 11434
  type: ClusterIP
```

Apply:

```bash
kubectl apply -f ollama.yaml
```

Wait until:

```bash
kubectl get pods -n kagent
```

Ollama should be `Running`.

---

# 🟢 STEP 5 — Pull Local Model

For 4GB RAM (recommended):

```bash
kubectl exec -it -n kagent deploy/ollama -- ollama pull tinyllama
```

Verify:

```bash
kubectl exec -it -n kagent deploy/ollama -- ollama list
```

You must see:

```
tinyllama:latest
```

---

# 🟢 STEP 6 — Create Ollama ModelConfig

Create:

```bash
nano ollama-model.yaml
```

Paste:

```yaml
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: ollama-model-config
  namespace: kagent
spec:
  model: tinyllama:latest
  provider: Ollama
  ollama: {}
```

Apply:

```bash
kubectl apply -f ollama-model.yaml
```

Restart controller:

```bash
kubectl rollout restart deployment kagent-controller -n kagent
```

---

# 🟢 STEP 7 — Access UI

```bash
kubectl port-forward -n kagent svc/kagent-ui 8080:8080
```

Open:

```
http://localhost:8080
```

---

# 🟢 STEP 8 — Create Agent

Create new agent.

Select:

```
Ollama (tinyllama:latest)
```

Type:

```
hi
```

It should respond.

---

# 🎉 Final Result

You now have:

* Fully Local AI
* No OpenAI
* No Gemini
* No quota limits
* No billing
* 100% inside Kubernetes

---

# 🛠 If Something Fails

Check:

```bash
kubectl logs -n kagent deploy/ollama
```

Check controller:

```bash
kubectl logs -n kagent deploy/kagent-controller
```

Test manually:

```bash
kubectl run debug --rm -it \
  --image=curlimages/curl \
  --namespace kagent \
  --restart=Never \
  --command -- sh
```

Inside:

```bash
curl -X POST http://ollama:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{"model":"tinyllama:latest","messages":[{"role":"user","content":"hello"}]}'
```

---
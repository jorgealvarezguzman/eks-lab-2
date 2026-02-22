# eks-lab-2
A service is up but requests time out — where do you look first in k8s?

> Key note for EKS: **NetworkPolicy is not enforced unless you have a policy engine** (Calico/Cilium) or AWS’s network-policy feature enabled. This lab uses **Calico installed via Helm** so the “service up but timeout” scenario is real.

---

## Lab goal

Practice answering/debugging:

> **A service is up but requests time out — where do you look first in k8s?**

You’ll reproduce a timeout from a client namespace, then debug it with a clean checklist:
**DNS → Service/Endpoints → Pod IP → NetworkPolicy**.

---

# 0) Prereqs

* AWS CLI configured
* Terraform >= 1.5
* kubectl
* Helm v3

---

# 1) Create EKS with Terraform (same as before)

Use your existing Terraform EKS module setup (VPC + EKS). After `terraform apply`, do:

```bash
aws eks update-kubeconfig --region us-east-1 --name eks-timeout-lab
kubectl get nodes
```

---

# 2) Install Calico (NetworkPolicy enforcement) via Helm

```bash
helm repo add projectcalico https://docs.tigera.io/calico/charts
helm repo update

helm install calico projectcalico/tigera-operator \
  --namespace tigera-operator --create-namespace
```

Wait until calico components are ready (this can take a bit):

```bash
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system 2>/dev/null || true
```

> If you don’t see `calico-system`, that’s ok—some installs name namespaces differently. The key is that Calico nodes/agents become ready. If Calico is not fully installed, NetworkPolicies won’t block traffic.

---

# 3) Helm chart 1: `echo-app` (service you’ll debug)

## Create chart

```bash
helm create echo-app
```

Replace files with the minimal versions below.

### `echo-app/values.yaml`

```yaml
replicaCount: 2

image:
  repository: ealen/echo-server
  tag: latest

service:
  port: 80

networkPolicy:
  enabled: false
  # When enabled, allow ingress only from namespaces listed here.
  allowFromNamespaces:
    - app
```

### `echo-app/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "echo-app.fullname" . }}
  labels:
    {{- include "echo-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "echo-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "echo-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: echo
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 80
```

### `echo-app/templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "echo-app.fullname" . }}
  labels:
    {{- include "echo-app.labels" . | nindent 4 }}
spec:
  type: ClusterIP
  selector:
    {{- include "echo-app.selectorLabels" . | nindent 4 }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
      protocol: TCP
      name: http
```

### `echo-app/templates/networkpolicy.yaml`

```yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "echo-app.fullname" . }}-ingress
spec:
  podSelector:
    matchLabels:
      {{- include "echo-app.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
  ingress:
    {{- range .Values.networkPolicy.allowFromNamespaces }}
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ . | quote }}
    {{- end }}
{{- end }}
```

### Install into `app` namespace

```bash
helm install echo ./echo-app -n app --create-namespace
kubectl -n app get deploy,pod,svc
```

---

# 4) Helm chart 2: `netshoot-client` (debug pod)

## Create chart

```bash
helm create netshoot-client
```

### `netshoot-client/values.yaml`

```yaml
replicaCount: 1

image:
  repository: nicolaka/netshoot
  tag: latest
```

### `netshoot-client/templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "netshoot-client.fullname" . }}
  labels:
    {{- include "netshoot-client.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "netshoot-client.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "netshoot-client.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: netshoot
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["sleep", "360000"]
```

Install into `client` namespace:

```bash
helm install client-tools ./netshoot-client -n client --create-namespace
kubectl -n client get pod
```

---

# 5) Baseline test (works)

Get the echo service DNS name:

* Release name = `echo`
* Chart fullname usually becomes `echo-echo-app` (depends on templates)

Run:

```bash
kubectl -n app get svc
```

Use the service name you see, for example `echo-echo-app`.

Then from client:

```bash
kubectl -n client exec -it deploy/client-tools-netshoot-client -- \
  curl -v --max-time 3 http://echo-echo-app.app.svc.cluster.local
```

✅ Should respond.

---

# 6) Create the timeout (via Helm upgrade enabling NetworkPolicy)

Enable NetworkPolicy allowing only `app` namespace (blocks `client` → timeout).

```bash
helm upgrade echo ./echo-app -n app \
  --set networkPolicy.enabled=true \
  --set networkPolicy.allowFromNamespaces[0]=app
```

Now test again from client:

```bash
kubectl -n client exec -it deploy/client-tools-netshoot-client -- \
  curl -v --connect-timeout 2 --max-time 3 http://echo-echo-app.app.svc.cluster.local
```

✅ Expected: **timeout**
And importantly: pods + service still look “up”.

---

# 7) Your debugging drill (do this in this order)

### A) DNS first

```bash
kubectl -n client exec -it deploy/client-tools-netshoot-client -- \
  nslookup echo-echo-app.app.svc.cluster.local
```

### B) Service → Endpoints (is traffic routed to pods?)

```bash
kubectl -n app get svc echo-echo-app -o wide
kubectl -n app describe svc echo-echo-app
kubectl -n app get endpoints echo-echo-app
kubectl -n app get endpointslice -l kubernetes.io/service-name=echo-echo-app
```

### C) Pod IP test (bypass Service)

```bash
POD_IP=$(kubectl -n app get pod -l app.kubernetes.io/instance=echo -o jsonpath='{.items[0].status.podIP}')
kubectl -n client exec -it deploy/client-tools-netshoot-client -- \
  curl -v --max-time 3 http://$POD_IP
```

### D) NetworkPolicy check (this is the “aha”)

```bash
kubectl get netpol -A
kubectl -n app describe netpol
```

Your interview line:

> “DNS resolves, service exists, endpoints exist, pod is ready—but cross-namespace traffic times out. Next I check NetworkPolicies (and CNI/policy engine) because this looks like a blocked path, not an app crash.”

---

# 8) Fix it (allow `client` namespace)

```bash
helm upgrade echo ./echo-app -n app \
  --set networkPolicy.enabled=true \
  --set networkPolicy.allowFromNamespaces[0]=app \
  --set networkPolicy.allowFromNamespaces[1]=client
```

Retest:

```bash
kubectl -n client exec -it deploy/client-tools-netshoot-client -- \
  curl -v --max-time 3 http://echo-echo-app.app.svc.cluster.local
```

✅ Works again.

---

# 9) Cleanup

```bash
helm uninstall echo -n app
helm uninstall client-tools -n client
helm uninstall calico -n tigera-operator

terraform destroy -auto-approve
```

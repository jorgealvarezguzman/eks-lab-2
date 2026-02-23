# eks-lab-2

> **“A service is up but requests time out—where do you look first in k8s?”**

You’ll build a small EKS cluster, deploy a simple “echo” service, then **intentionally create a timeout** (via a NetworkPolicy) and practice a **repeatable debug flow**.

---

## The “where do you look first?” answer

When you hear **timeout**, assume **network path / routing / policy / LB / DNS** first (not app bugs).

**Order of attack (fastest isolation):**

1. **Reproduce + identify what kind of timeout**

* Is it **connect timeout** (can’t reach service) vs **read timeout** (connected, app slow)?
* `curl -v` shows where it hangs.

2. **Test from inside the cluster**

* If it times out **from inside too**, it’s Kubernetes networking/policy/service/pods.
* If it only times out **from outside**, it’s Ingress/LB/Security Groups/target groups/routing.

3. **Check Service → Endpoints → Pods**

* `kubectl get svc,ep,endpointslice -n <ns>`
* `kubectl describe svc <svc> -n <ns>`
* Endpoints missing or wrong port = you found it.

4. **Check NetworkPolicies**

* `kubectl get netpol -A`
* A default-deny or missing allow rule is a *perfect* “service up, timeout” cause.

5. **Then go deeper (if needed)**

* kube-dns/CoreDNS, CNI (aws-vpc-cni), node security groups, NACLs, kube-proxy/IPTables, etc.

This lab makes you practice steps **2–4** hard.

---

# Lab: Terraform + AWS EKS + “timeout” scenario

## Prereqs (local)

* AWS CLI configured (`aws sts get-caller-identity` works)
* Terraform >= 1.5
* kubectl
* (optional but recommended) Helm

> **Cost note:** EKS control plane + nodes cost money. Destroy when done.

---

## 1) Terraform: create EKS (us-east-1)

Create folder `eks-timeout-lab/` with:

### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### `variables.tf`

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "cluster_name" {
  type    = string
  default = "eks-timeout-lab"
}
```

### `main.tf`

```hcl
provider "aws" {
  region = var.region
}

data "aws_availability_zones" "available" {}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = local.azs
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  # Required tags so EKS can create LBs in these subnets if you later test LoadBalancers/Ingress
  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.29"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  enable_irsa = true

  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.medium"]
      min_size       = 2
      max_size       = 3
      desired_size   = 2
    }
  }
}

output "cluster_name" {
  value = module.eks.cluster_name
}
```

### Apply

```bash
cd eks-timeout-lab
terraform init
terraform apply -auto-approve
```

### Configure kubectl

```bash
aws eks update-kubeconfig --region us-east-1 --name eks-timeout-lab
kubectl get nodes
```

---

## 2) Deploy a simple service + two client pods

Create `k8s/app.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
---
apiVersion: v1
kind: Namespace
metadata:
  name: client
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  namespace: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: ealen/echo-server:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: echo
  namespace: app
spec:
  selector:
    app: echo
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Pod
metadata:
  name: netshoot-app
  namespace: app
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot:latest
    command: ["sleep", "360000"]
---
apiVersion: v1
kind: Pod
metadata:
  name: netshoot-client
  namespace: client
spec:
  containers:
  - name: netshoot
    image: nicolaka/netshoot:latest
    command: ["sleep", "360000"]
```

Apply:

```bash
kubectl apply -f k8s/app.yaml
kubectl -n app get pod,svc
kubectl -n client get pod
```

### Confirm it works (baseline)

From **client namespace**, call the service:

```bash
kubectl -n client exec -it netshoot-client -- \
  curl -v --max-time 3 http://echo.app.svc.cluster.local
```

You should get a response.

---

## 3) Inject the “service is up but requests time out” failure (NetworkPolicy)

Create `k8s/deny-cross-ns.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: echo
  policyTypes:
  - Ingress
  ingress:
  # Only allow traffic from pods in the SAME namespace that have label allow=true
  - from:
    - podSelector:
        matchLabels:
          allow: "true"
```

Apply it:

```bash
kubectl apply -f k8s/deny-cross-ns.yaml
```

Now test again from **client**:

```bash
kubectl -n client exec -it netshoot-client -- \
  curl -v --connect-timeout 2 --max-time 3 http://echo.app.svc.cluster.local
```

✅ Expected: **timeout** (the service/pods still look “up”, but traffic is blocked).

---

# Debug drill

## Step 1 — Is it DNS?

```bash
kubectl -n client exec -it netshoot-client -- nslookup echo.app.svc.cluster.local
```

If DNS fails → check CoreDNS pods/logs.

## Step 2 — Is the Service wired correctly?

```bash
kubectl -n app get svc echo -o wide
kubectl -n app describe svc echo
kubectl -n app get endpoints echo
kubectl -n app get endpointslice -l kubernetes.io/service-name=echo
```

If endpoints empty → label mismatch, readiness, selector wrong.

## Step 3 — Can I reach the pods directly?

Get a pod IP and curl it:

```bash
POD_IP=$(kubectl -n app get pod -l app=echo -o jsonpath='{.items[0].status.podIP}')
kubectl -n client exec -it netshoot-client -- curl -v --max-time 3 http://$POD_IP
```

If pod IP also times out → policy/CNI/security group.

## Step 4 — Are NetworkPolicies blocking?

```bash
kubectl get netpol -A
kubectl -n app describe netpol deny-from-other-namespaces
```

**Punchline:** “Service and endpoints exist, pods are ready, DNS resolves, but cross-namespace traffic times out → check NetworkPolicy.”

---

## Fix it (add an allow rule)

Label the client pod to allow it (or adjust policy). Quick fix: label the client pod and allow same-namespace only won’t help—policy is in `app` namespace. Let’s allow traffic from namespace `client`.

Create `k8s/allow-client-ns.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-client-namespace
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: echo
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: client
```

Apply:

```bash
kubectl apply -f k8s/allow-client-ns.yaml
kubectl -n client exec -it netshoot-client -- curl -v --max-time 3 http://echo.app.svc.cluster.local
```

✅ Requests should work again.

---

# Optional: 2 extra “timeouts” you can practice

## A) “Endpoints missing” (Service exists, but no pods match selector)

Edit service selector to `app: echo-typo` and observe:

```bash
kubectl -n app get endpoints echo
```

This often yields **connection errors** rather than timeouts, but it’s still a must-check.

## B) “App-level slow/hung” (read timeout)

Swap the app with something that sleeps before responding (or add a sidecar that delays). Then `curl` connects but hangs on response → you shift from networking to app perf + downstream dependency tracing.

---

## Cleanup (don’t forget)

```bash
terraform destroy -auto-approve
```

---

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
  --set 'networkPolicy.allowFromNamespaces[0]=app'
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

> “DNS resolves, service exists, endpoints exist, pod is ready—but cross-namespace traffic times out. Next I check NetworkPolicies (and CNI/policy engine) because this looks like a blocked path, not an app crash.”

---

# 8) Fix it (allow `client` namespace)

```bash
helm upgrade echo ./echo-app -n app \                              
  --set networkPolicy.enabled=true \                             
  --set 'networkPolicy.allowFromNamespaces[0]=app' \
  --set 'networkPolicy.allowFromNamespaces[1]=client'
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

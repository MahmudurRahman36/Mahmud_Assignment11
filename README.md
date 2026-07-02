# Module 11 Assignment – Kubernetes Application Deployment & Management
**Submitted by:** Mahmudur Rahman  
**Date:** July 2, 2026  
**Cluster Tool:** Minikube v1.38.1 on AWS EC2 (t3.medium, Ubuntu 22.04, ap-south-1)  
**Kubernetes Version:** v1.35.1

---

## Part 1: Conceptual Understanding

### Q1. How does Kubernetes ensure high availability compared to traditional deployment?

Kubernetes ensures high availability through several built-in mechanisms. It runs multiple replicas of an application simultaneously across different nodes, so if one pod or node fails, traffic is automatically rerouted to healthy pods. The **ReplicaSet controller** continuously monitors the desired replica count and recreates any failed pods without manual intervention. Kubernetes also performs **health checks** (liveness and readiness probes) to detect and restart unhealthy containers. In contrast, traditional deployments run on a single or small set of servers with no automated failover — a crash means manual intervention and downtime.

---

### Q2. Explain the relationship between Pods, ReplicaSets, and Deployments.

A **Pod** is the smallest deployable unit in Kubernetes — it wraps one or more containers that share network and storage. A **ReplicaSet** manages a group of identical Pods, ensuring a specified number of replicas are always running; it will create or delete Pods to match the desired count. A **Deployment** sits above the ReplicaSet and manages its lifecycle — it creates ReplicaSets, handles rolling updates by creating a new ReplicaSet while scaling down the old one, and supports rollbacks. In practice, users define Deployments, which then create and manage ReplicaSets, which in turn create and manage Pods.

---

### Q3. Why are Services required in Kubernetes?

Pods in Kubernetes are **ephemeral** — they can be created, destroyed, and assigned new IP addresses at any time. Without a Service, clients would need to track changing Pod IPs, which is impractical. A **Service** provides a stable virtual IP (ClusterIP) and DNS name that remains constant regardless of Pod restarts. It also acts as a **load balancer**, distributing traffic evenly across all matching Pods using label selectors. Without Services, there would be no reliable way for components inside or outside the cluster to communicate with application Pods.

---

### Q4. Difference between ConfigMaps and Secrets with a practical example.

Both ConfigMaps and Secrets store configuration data outside container images, but they serve different purposes:

| | ConfigMap | Secret |
|---|---|---|
| **Purpose** | Non-sensitive config | Sensitive credentials |
| **Storage** | Plain text | Base64 encoded (encrypted at rest) |
| **Use case** | App settings, feature flags | Passwords, API keys, TLS certs |

**Practical Example:**
- **ConfigMap** stores `APP_MODE=dev`, `APP_ENV=development`, `APP_VERSION=1.0` — these are non-sensitive environment settings.
- **Secret** stores `DB_USERNAME=admin` and `DB_PASSWORD=P@ssw0rd123` in base64 encoding — these are credentials that must not be exposed in plain text.

Both are injected into the Pod as environment variables, keeping the container image clean and portable.

---

## Part 2: Cluster Setup & Verification

### Cluster Setup

Set up a Kubernetes cluster using **Minikube** on an AWS EC2 `t3.medium` instance running Ubuntu 22.04.

**Installation Steps:**
```bash
# Install Docker
sudo apt-get update && sudo apt-get install -y docker.io
sudo systemctl enable docker && sudo usermod -aG docker ubuntu

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo mv kubectl /usr/local/bin/

# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start --driver=docker --memory=3500 --cpus=2
```

### Node Verification

```
$ kubectl get nodes -o wide

NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
minikube   Ready    control-plane   43s   v1.35.1   192.168.49.2   <none>        Debian GNU/Linux 12 (bookworm)   6.8.0-1060-aws   docker://29.2.1

$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### System Pods in kube-system Namespace

```
$ kubectl get pods -n kube-system

NAME                               READY   STATUS    RESTARTS   AGE
coredns-7d764666f9-k8tmq           1/1     Running   0          2m35s
etcd-minikube                      1/1     Running   0          2m40s
kube-apiserver-minikube            1/1     Running   0          2m40s
kube-controller-manager-minikube   1/1     Running   0          2m40s
kube-proxy-vmlq5                   1/1     Running   0          2m36s
kube-scheduler-minikube            1/1     Running   0          2m40s
storage-provisioner                1/1     Running   1 (4s ago) 2m38s
```

**Screenshots:** `screenshots/01_cluster_nodes.png`, `screenshots/02_kube_system_pods.png`

### Observation

The Minikube cluster runs a single-node setup where the same node acts as both the **control plane and worker**. The `kube-system` namespace hosts core Kubernetes components: `etcd` (cluster state store), `kube-apiserver` (API gateway), `kube-controller-manager` (manages controllers like ReplicaSet), `kube-scheduler` (assigns pods to nodes), `kube-proxy` (manages network rules), and `coredns` (provides DNS for service discovery). The `storage-provisioner` is a Minikube-specific addon that handles dynamic volume provisioning.

---

## Part 3: Multi-Resource Deployment

### Resources Deployed

- **Deployment:** `nginx-deployment` — 2 replicas, image `nginx:alpine`, labels `app: nginx`
- **Service:** `nginx-service` — NodePort type, port 80 → nodePort 30007

### Commands

```bash
kubectl apply -f 01-configmap.yaml  # configmap/nginx-config created
kubectl apply -f 02-secret.yaml     # secret/nginx-secret created
kubectl apply -f 03-deployment.yaml # deployment.apps/nginx-deployment created
kubectl apply -f 04-service.yaml    # service/nginx-service created
```

### Verification

```
$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-deployment-58ff9bb9f7-gbrhc   1/1     Running   0          14s   10.244.0.3   minikube
nginx-deployment-58ff9bb9f7-w8b2r   1/1     Running   0          14s   10.244.0.4   minikube

$ kubectl get deployment nginx-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           30s

$ kubectl get service nginx-service
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-service   NodePort   10.107.23.154   <none>        80:30007/TCP   30s
```

### Browser Access

Nginx welcome page was accessible at `http://3.110.161.219:30007` via EC2 public IP and NodePort.

**Screenshots:** `screenshots/03_running_pods.png`, `screenshots/04_service_access_browser.png`

---

## Part 4: Configuration & Secrets

### ConfigMap (`01-configmap.yaml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  APP_MODE: "dev"
  APP_ENV: "development"
  APP_VERSION: "1.0"
```

### Secret (`02-secret.yaml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
type: Opaque
data:
  DB_USERNAME: YWRtaW4=         # base64: admin
  DB_PASSWORD: UEBzc3cwcmQxMjM= # base64: P@ssw0rd123
```

### Injection into Deployment

```yaml
envFrom:
  - configMapRef:
      name: nginx-config
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: nginx-secret
        key: DB_USERNAME
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: nginx-secret
        key: DB_PASSWORD
```

### Verification Inside Container

```
$ kubectl exec nginx-deployment-58ff9bb9f7-gbrhc -- env | grep -E "APP_MODE|APP_ENV|APP_VERSION|DB_USERNAME|DB_PASSWORD"

APP_VERSION=1.0
DB_USERNAME=admin
DB_PASSWORD=P@ssw0rd123
APP_ENV=development
APP_MODE=dev
```

All 5 environment variables correctly injected from ConfigMap and Secret.

**Screenshot:** `screenshots/05_configmap_secret_verify.png`

---

## Part 5: Scaling & Rolling Updates

### Scale to 4 Replicas

```bash
$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled

$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-58ff9bb9f7-2q79j   1/1     Running   0          2s
nginx-deployment-58ff9bb9f7-9drsv   1/1     Running   0          2s
nginx-deployment-58ff9bb9f7-gbrhc   1/1     Running   0          17s
nginx-deployment-58ff9bb9f7-w8b2r   1/1     Running   0          17s
```

**Screenshot:** `screenshots/06_scaled_pods.png`

### Rolling Update

```bash
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.25-alpine

$ kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 4 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 4 new replicas have been updated...
deployment "nginx-deployment" successfully rolled out
```

**Screenshot:** `screenshots/07_rollout_status.png`

### Rollback

```bash
$ kubectl rollout undo deployment/nginx-deployment
deployment.apps/nginx-deployment rolled back

$ kubectl rollout status deployment/nginx-deployment
deployment "nginx-deployment" successfully rolled out

$ kubectl rollout history deployment/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

**Screenshot:** `screenshots/08_rollback_done.png`

### Explanation

**Rolling Update:** Kubernetes did NOT stop all pods at once. It created a new ReplicaSet with `nginx:1.25-alpine` and gradually replaced old pods with new ones — one at a time. With `maxSurge: 1` and `maxUnavailable: 0`, there was never a period where fewer than 4 pods were serving traffic. The application remained fully available throughout the update.

**Rollback:** `kubectl rollout undo` restored the previous ReplicaSet (`nginx:alpine`). Kubernetes used the same rolling strategy in reverse — scaling the old ReplicaSet back up while scaling down the new one. All 4 replicas reverted to the original image without any service disruption.

---

## Part 6: Basic Troubleshooting

### Step 1 — Intentionally Breaking the Deployment

```bash
$ kubectl set image deployment/nginx-deployment nginx=nginx:broken-image-xyz-notexist
deployment.apps/nginx-deployment image updated
```

### Step 2 — Observing Pod Status

```
$ kubectl get pods -o wide

NAME                                READY   STATUS             RESTARTS   AGE
nginx-deployment-58ff9bb9f7-dw4zs   1/1     Running            0          22s
nginx-deployment-58ff9bb9f7-pjcr8   1/1     Running            0          23s
nginx-deployment-58ff9bb9f7-qjbpq   1/1     Running            0          26s
nginx-deployment-58ff9bb9f7-wcwqb   1/1     Running            0          24s
nginx-deployment-7685dd86fd-tkbgd   0/1     ImagePullBackOff   0          21s
```

**Screenshot:** `screenshots/09_broken_pod_status.png`

### Step 3 — kubectl describe pod

```bash
$ kubectl describe pod nginx-deployment-7685dd86fd-tkbgd

Events:
  Type     Reason     Age   Message
  ----     ------     ----  -------
  Normal   Pulling    14s   Pulling image "nginx:broken-image-xyz-notexist"
  Warning  Failed     14s   Failed to pull image: not found
  Warning  Failed     14s   Error: ErrImagePull
  Warning  BackOff    1s    Back-off pulling image "nginx:broken-image-xyz-notexist"
```

**Screenshot:** `screenshots/10_describe_broken_pod.png`

### Step 4 — Fix the Issue

```bash
$ kubectl set image deployment/nginx-deployment nginx=nginx:alpine
$ kubectl rollout status deployment/nginx-deployment
deployment "nginx-deployment" successfully rolled out

$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-58ff9bb9f7-dw4zs   1/1     Running   0          3m
nginx-deployment-58ff9bb9f7-pjcr8   1/1     Running   0          3m
nginx-deployment-58ff9bb9f7-qjbpq   1/1     Running   0          3m
nginx-deployment-58ff9bb9f7-wcwqb   1/1     Running   0          3m
```

**Screenshot:** `screenshots/11_fixed_running.png`

### Explanation: How Issue Was Identified and Resolved

**Identification:** The `kubectl get pods` command immediately showed the new pod stuck in **`ImagePullBackOff`** — Kubernetes could not pull the container image. Running `kubectl describe pod` confirmed this in the Events section: the message `Failed to pull image "nginx:broken-image-xyz-notexist": not found` pinpointed the exact cause. The `BackOff` state means Kubernetes was retrying the pull with exponentially increasing delays. Importantly, the rolling update strategy protected the existing 4 healthy pods from being terminated — they kept serving traffic while the broken pod was stuck.

**Resolution:** The incorrect image name was fixed by running `kubectl set image` with the valid image `nginx:alpine`. This triggered a new rolling update that terminated the broken pod and confirmed all 4 replicas were running the correct image.

---

## Part 7: Namespaces (Isolation)

### Create Namespace and Deploy Resources

```bash
$ kubectl apply -f 05-namespace-deployment.yaml

namespace/dev-env created
configmap/nginx-config created
secret/nginx-secret created
deployment.apps/nginx-deployment created
service/nginx-service created
```

### Resources in dev-env Namespace

```
$ kubectl get all -n dev-env

NAME                                   READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-745d4b6b6-7jbmt   1/1     Running   0          2s
pod/nginx-deployment-745d4b6b6-t2s64   1/1     Running   0          2s

NAME                    TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
service/nginx-service   NodePort   10.103.5.6   <none>        80:30008/TCP   2s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   2/2     2            2           2s
```

**Screenshot:** `screenshots/12_dev_env_namespace.png`

### Isolation Demonstration

```
$ kubectl get pods --all-namespaces

NAMESPACE     NAME                                   READY   STATUS
default       nginx-deployment-58ff9bb9f7-dw4zs      1/1     Running
default       nginx-deployment-58ff9bb9f7-pjcr8      1/1     Running
default       nginx-deployment-58ff9bb9f7-qjbpq      1/1     Running
default       nginx-deployment-58ff9bb9f7-wcwqb      1/1     Running
dev-env       nginx-deployment-745d4b6b6-7jbmt       1/1     Running
dev-env       nginx-deployment-745d4b6b6-t2s64       1/1     Running
kube-system   coredns-7d764666f9-k8tmq               1/1     Running
kube-system   etcd-minikube                          1/1     Running
...
```

The `default` namespace pods and `dev-env` namespace pods are completely isolated. The `nginx-service` in `dev-env` uses NodePort `30008` (vs `30007` in default), and the two environments cannot access each other's services without explicit cross-namespace DNS (`service.namespace.svc.cluster.local`).

**Screenshot:** `screenshots/13_default_namespace_isolation.png`

---

## Submission Checklist

| Requirement | Status | File |
|---|---|---|
| ConfigMap YAML | ✅ | `manifests/01-configmap.yaml` |
| Secret YAML | ✅ | `manifests/02-secret.yaml` |
| Deployment YAML | ✅ | `manifests/03-deployment.yaml` |
| Service YAML | ✅ | `manifests/04-service.yaml` |
| Namespace Deployment YAML | ✅ | `manifests/05-namespace-deployment.yaml` |
| Running Pods screenshot | ✅ | `screenshots/03_running_pods.png` |
| Browser access screenshot | ✅ | `screenshots/04_service_access_browser.png` |
| Scaling result screenshot | ✅ | `screenshots/06_scaled_pods.png` |
| Conceptual answers (Part 1) | ✅ | This document |
| Cluster observations (Part 2) | ✅ | This document |
| Update/rollback explanation (Part 5) | ✅ | This document |
| Troubleshoot explanation (Part 6) | ✅ | This document |

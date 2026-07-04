# Module 11 Assignment – Kubernetes Application Deployment & Management

**Submitted by:** Mahmudur Rahman
**Cluster tool:** Minikube (docker driver) on an AWS EC2 instance (t3.medium, Ubuntu 22.04, ap-south-1 / Mumbai)
**Kubernetes version:** v1.35.1

---

## Part 1: Conceptual Understanding

### Q1. How does Kubernetes ensure high availability compared to traditional deployment?

In a traditional deployment, we usually run the app on one or two servers, and if that server goes down, the app goes down with it, and someone has to manually restart it. Kubernetes solves this problem differently. It always tries to keep a certain number of copies (replicas) of our app running, and this number is controlled by a Deployment/ReplicaSet. If one Pod crashes or the node it was running on fails, Kubernetes notices this immediately and creates a new Pod to replace it, without any human action needed. It also spreads Pods across different nodes when possible, so a single node failure does not take down the whole app. On top of that, Kubernetes runs health checks (liveness/readiness probes) so it can detect a broken container and restart it automatically. This self-healing behaviour is the main reason Kubernetes gives much better availability than a normal single-server setup.

### Q2. Explain the relationship between Pods, ReplicaSets, and Deployments.

These three are layered on top of each other. A **Pod** is the smallest unit in Kubernetes, it is basically one or more containers running together with shared network/storage. A **ReplicaSet** is the thing that makes sure a fixed number of identical Pods are running at all times, if a Pod dies, the ReplicaSet creates a new one to keep the count correct. A **Deployment** sits on top of the ReplicaSet, we normally do not create ReplicaSets by hand, we create a Deployment, and the Deployment creates and manages the ReplicaSet for us. The Deployment is also what gives us rolling updates and rollbacks, when we change the image version, the Deployment creates a new ReplicaSet and slowly shifts traffic from old Pods to new Pods. So in short: Deployment manages ReplicaSet, and ReplicaSet manages Pods.

### Q3. Why are Services required in Kubernetes?

Pods are not permanent, they can restart, get rescheduled to another node, or get replaced during a rolling update, and every time this happens the Pod gets a new IP address. If other parts of our app (or users outside the cluster) had to remember these changing Pod IPs, nothing would work reliably. A **Service** gives us one stable address (a ClusterIP and a DNS name) that never changes, even though the Pods behind it keep changing. The Service also does basic load balancing, it sends incoming traffic to whichever Pods currently match its label selector. Without a Service, we would have no dependable way to reach our Pods from inside or outside the cluster.

### Q4. Difference between ConfigMaps and Secrets with a practical example.

Both are used to keep configuration values outside of our container image, so we don't have to rebuild the image every time a setting changes. The difference is what kind of data they are meant for.

| | ConfigMap | Secret |
|---|---|---|
| Meant for | normal, non-sensitive settings | sensitive data like passwords, tokens |
| How it is stored | plain text | base64 encoded (not real encryption by default) |
| Example | `APP_MODE=dev` | `DB_PASSWORD=P@ssw0rd123` |

In this assignment, I created a ConfigMap called `nginx-config` that stores `APP_MODE=dev`, `APP_ENV=development`, `APP_VERSION=1.0`. I also created a Secret called `nginx-secret` that stores a dummy `DB_USERNAME` and `DB_PASSWORD`. Both were injected into the same Deployment as environment variables, so the container image itself never had to contain any of this information.

---

## Part 2: Cluster Setup & Verification

### How I set up the cluster

I did this whole assignment on an AWS EC2 instance instead of my local PC, because it was easier to keep it running and it also matches how we would normally do this on a real project. Steps I followed on the EC2 instance (Ubuntu 22.04):

```bash
# install docker
sudo apt-get update && sudo apt-get install -y docker.io
sudo systemctl enable docker --now
sudo usermod -aG docker ubuntu

# install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo mv kubectl /usr/local/bin/

# install minikube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube /usr/local/bin/minikube

# start the cluster
minikube start --driver=docker --memory=3500 --cpus=2
```

### Node verification

```
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION   CONTAINER-RUNTIME
minikube   Ready    control-plane   38m   v1.35.1   192.168.49.2   <none>        Debian GNU/Linux 12 (bookworm)   6.8.0-1060-aws   docker://29.2.1

Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Screenshot: `screenshots/01_cluster_nodes.png`

### kube-system pods

```
NAME                               READY   STATUS    RESTARTS      AGE
coredns-7d764666f9-vwtb5           1/1     Running   0             39m
etcd-minikube                      1/1     Running   0             39m
kube-apiserver-minikube            1/1     Running   0             39m
kube-controller-manager-minikube   1/1     Running   0             39m
kube-proxy-gpm4f                   1/1     Running   0             39m
kube-scheduler-minikube            1/1     Running   0             39m
storage-provisioner                1/1     Running   1 (38m ago)   39m
```

Screenshot: `screenshots/02_kube_system_pods.png`

### What I observed

Minikube gives us a single node that acts as both control-plane and worker at the same time, that's why we only see one node in `kubectl get nodes`. Inside `kube-system` we can see all the core Kubernetes components running as their own Pods: `etcd` (stores the cluster's data), `kube-apiserver` (the front door for all kubectl commands), `kube-controller-manager` and `kube-scheduler` (these keep the desired state and decide where Pods go), `kube-proxy` (handles the networking rules), `coredns` (internal DNS for service discovery), and `storage-provisioner` which is added by Minikube itself to handle storage. Seeing all of these already Running right after `minikube start` tells us the control plane came up correctly.

---

## Part 3: Multi-Resource Deployment

I deployed Nginx using a Deployment with 2 replicas and exposed it with a NodePort Service.

- Deployment: `nginx-deployment`, image `nginx:alpine`, 2 replicas, labels `app: nginx`
- Service: `nginx-service`, type NodePort, port 80 → nodePort 30007

```bash
kubectl apply -f 01-configmap.yaml
kubectl apply -f 02-secret.yaml
kubectl apply -f 03-deployment.yaml
kubectl apply -f 04-service.yaml
```

### Verification

```
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-deployment-67659d477d-7r2cc   1/1     Running   0          17m   10.244.0.13   minikube
nginx-deployment-67659d477d-nbpjn   1/1     Running   0          17m   10.244.0.12   minikube

NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           23m

NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service   NodePort   10.98.79.223   <none>        80:30007/TCP   23m
```

Screenshot: `screenshots/03_running_pods.png`

### Accessing it from the browser

One thing I ran into here: since Minikube was running with the `docker` driver, the NodePort is only open on Minikube's own internal IP (`192.168.49.2`), not directly on the EC2 machine's public IP. To make the app reachable from my laptop's browser, I ran:

```bash
kubectl port-forward --address 0.0.0.0 svc/nginx-service 30007:80
```

After that, opening `http://<EC2-public-ip>:30007` in Chrome showed the Nginx welcome page.

Screenshot: `screenshots/04_service_access_browser.png`

---

## Part 4: Configuration & Secrets

### ConfigMap (`manifests/01-configmap.yaml`)

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

### Secret (`manifests/02-secret.yaml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
type: Opaque
data:
  DB_USERNAME: YWRtaW4=        # admin
  DB_PASSWORD: UEBzc3cwcmQxMjM= # P@ssw0rd123
```

### How it is injected into the Deployment

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

### Verified inside the container

```
$ kubectl exec nginx-deployment-67659d477d-7r2cc -- env | grep -E 'APP_MODE|APP_ENV|APP_VERSION|DB_USERNAME|DB_PASSWORD'
APP_ENV=development
APP_MODE=dev
APP_VERSION=1.0
DB_USERNAME=admin
DB_PASSWORD=P@ssw0rd123
```

All 5 values from the ConfigMap and Secret showed up correctly inside the running container.

Screenshot: `screenshots/05_configmap_secret_verify.png`

---

## Part 5: Scaling & Rolling Updates

### Scaling to 4 replicas

```
$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled

NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-67659d477d-4rpqh   1/1     Running   0          20s
nginx-deployment-67659d477d-7r2cc   1/1     Running   0          23m
nginx-deployment-67659d477d-lq6mh   1/1     Running   0          19s
nginx-deployment-67659d477d-nbpjn   1/1     Running   0          23m
```

Screenshot: `screenshots/06_scaled_pods.png`

### Rolling update

I changed the image from `nginx:alpine` to `nginx:1.25-alpine`:

```
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.25-alpine
$ kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 4 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 4 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out

$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP
nginx-deployment-6bc475b855-dzdfr   1/1     Running   0          19s   10.244.0.23
nginx-deployment-6bc475b855-gvbwj   1/1     Running   0          20s   10.244.0.22
nginx-deployment-6bc475b855-h6tbb   1/1     Running   0          21s   10.244.0.21
nginx-deployment-6bc475b855-sn9l6   1/1     Running   0          23s   10.244.0.20
```

Screenshot: `screenshots/07_rollout_status.png`

### Rollback

```
$ kubectl rollout undo deployment/nginx-deployment
deployment.apps/nginx-deployment rolled back

$ kubectl rollout status deployment/nginx-deployment
deployment "nginx-deployment" successfully rolled out

$ kubectl rollout history deployment/nginx-deployment
REVISION  CHANGE-CAUSE
4         <none>
6         <none>
7         <none>

$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP
nginx-deployment-67659d477d-5wh4b   1/1     Running   0          22s   10.244.0.24
nginx-deployment-67659d477d-cqk7d   1/1     Running   0          20s   10.244.0.25
nginx-deployment-67659d477d-d8bwc   1/1     Running   0          19s   10.244.0.26
nginx-deployment-67659d477d-g7b2x   1/1     Running   0          18s   10.244.0.27
```

Screenshot: `screenshots/08_rollback_done.png`

### What actually happened during the update and rollback

When I changed the image, Kubernetes did not kill all 4 Pods at once. Because my Deployment uses a rolling update strategy (`maxSurge: 1`, `maxUnavailable: 0`), it brought up new Pods with the new image one at a time, and only removed an old Pod once a new one was ready. This means the app never went fully down while it was updating, there were always at least 4 Pods able to serve traffic. Kubernetes did this by creating a brand new ReplicaSet for the new image version, and slowly scaling that one up while scaling the old ReplicaSet down.

For the rollback, `kubectl rollout undo` basically reversed the same process, it scaled the old ReplicaSet (with `nginx:alpine`) back up and scaled the newer one back down, again one Pod at a time so the app kept working the whole time. After the rollback finished, all 4 Pods were back on the original image, and I could confirm this by checking the Pod names again (they matched the very first ReplicaSet hash, `67659d477d`).

---

## Part 6: Basic Troubleshooting

### Step 1 – I broke the deployment on purpose

```
$ kubectl set image deployment/nginx-deployment nginx=nginx:broken-image-xyz-notexist
deployment.apps/nginx-deployment image updated
```

### Step 2 – Pod status right after

```
NAME                                READY   STATUS         RESTARTS   AGE
nginx-deployment-5f4c498b55-5hmjf   0/1     ErrImagePull   0          12s
nginx-deployment-67659d477d-5wh4b   1/1     Running        0          62s
nginx-deployment-67659d477d-cqk7d   1/1     Running        0          60s
nginx-deployment-67659d477d-d8bwc   1/1     Running        0          59s
nginx-deployment-67659d477d-g7b2x   1/1     Running        0          58s
```

Screenshot: `screenshots/09_broken_pod_status.png`

### Step 3 – kubectl describe pod

```
Events:
  Type     Reason   Age  From     Message
  ----     ------   ---- ----     -------
  Normal   Pulling  35s (x3 over 76s)  kubelet  Pulling image "nginx:broken-image-xyz-notexist"
  Warning  Failed   33s (x3 over 74s)  kubelet  Failed to pull image "nginx:broken-image-xyz-notexist": manifest unknown
  Warning  Failed   33s (x3 over 74s)  kubelet  Error: ErrImagePull
  Normal   BackOff  7s  (x4 over 73s)  kubelet  Back-off pulling image "nginx:broken-image-xyz-notexist"
  Warning  Failed   7s  (x4 over 73s)  kubelet  Error: ImagePullBackOff
```

Screenshot: `screenshots/10_describe_broken_pod.png`

### Step 4 – Fixing it

```
$ kubectl set image deployment/nginx-deployment nginx=nginx:alpine
$ kubectl rollout status deployment/nginx-deployment
deployment "nginx-deployment" successfully rolled out

NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           33m
```

Screenshot: `screenshots/11_fixed_running.png`

### How I found and fixed the problem

Right after I applied the bad image name, `kubectl get pods` showed the new Pod stuck in `ErrImagePull`, which quickly turned into `ImagePullBackOff`. This already told me Kubernetes could not download the image. To be sure, I ran `kubectl describe pod` on that Pod and looked at the Events section at the bottom, it clearly said `Failed to pull image "nginx:broken-image-xyz-notexist": manifest unknown`, so I knew straight away the image tag simply did not exist on Docker Hub. Because of the rolling update strategy, the 4 older, healthy Pods were never touched, so the app kept working while one broken Pod kept retrying in the background. To fix it, I just set the image back to `nginx:alpine`, which created a fresh rollout, replaced the broken Pod, and got all 4 replicas back to Running.

---

## Part 7: Namespaces (Isolation)

I created a new namespace called `dev-env` and deployed the same app (ConfigMap, Secret, Deployment, Service) inside it, using `manifests/05-namespace-deployment.yaml`.

```
$ kubectl apply -f 05-namespace-deployment.yaml
namespace/dev-env created
configmap/nginx-config created
secret/nginx-secret created
deployment.apps/nginx-deployment created
service/nginx-service created
```

### Resources inside dev-env

```
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-67659d477d-pgwhq   1/1     Running   0          24m
pod/nginx-deployment-67659d477d-v9gfm   1/1     Running   0          24m

NAME                    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/nginx-service   NodePort   10.111.106.161   <none>        80:30008/TCP   24m

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   2/2     2            2           24m
```

Screenshot: `screenshots/12_dev_env_namespace.png`

### Proof of isolation

```
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                READY   STATUS    RESTARTS
default       nginx-deployment-67659d477d-5wh4b   1/1     Running   0
default       nginx-deployment-67659d477d-cqk7d   1/1     Running   0
default       nginx-deployment-67659d477d-d8bwc   1/1     Running   0
default       nginx-deployment-67659d477d-g7b2x   1/1     Running   0
dev-env       nginx-deployment-67659d477d-pgwhq   1/1     Running   0
dev-env       nginx-deployment-67659d477d-v9gfm   1/1     Running   0
kube-system   coredns-7d764666f9-vwtb5            1/1     Running   0
...
```

Even though both `default` and `dev-env` have a Deployment and a Service with the exact same names (`nginx-deployment`, `nginx-service`), Kubernetes keeps them completely separate, they don't clash with each other. The proof of this is that both got different NodePorts assigned (30007 for `default`, 30008 for `dev-env`) and different ClusterIPs, and running `kubectl get pods` without `-n` only shows the `default` namespace Pods, the `dev-env` ones don't show up unless I explicitly ask for that namespace. This is exactly what namespaces are for, keeping different environments (like dev vs staging vs prod) isolated on the same cluster.

Screenshot: `screenshots/13_default_namespace_isolation.png`

---

## Submission Checklist

| Requirement | Status | File |
|---|---|---|
| ConfigMap YAML | Done | `manifests/01-configmap.yaml` |
| Secret YAML | Done | `manifests/02-secret.yaml` |
| Deployment YAML | Done | `manifests/03-deployment.yaml` |
| Service YAML | Done | `manifests/04-service.yaml` |
| Namespace + Deployment YAML | Done | `manifests/05-namespace-deployment.yaml` |
| Running Pods screenshot | Done | `screenshots/03_running_pods.png` |
| Browser access screenshot | Done | `screenshots/04_service_access_browser.png` |
| Scaling result screenshot | Done | `screenshots/06_scaled_pods.png` |
| Conceptual answers (Part 1) | Done | This document |
| Cluster observations (Part 2) | Done | This document |
| Update/rollback explanation (Part 5) | Done | This document |
| Troubleshooting explanation (Part 6) | Done | This document |
| Namespace isolation (Part 7) | Done | This document |

# Kubernetes MCP Server on Minikube

This tutorial builds a small local Kubernetes lab, installs the Kubernetes MCP Server with Helm, connects it to VS Code, and uses an LLM to diagnose and repair a deliberately broken nginx workload. It is written for Ubuntu on WSL, but the Kubernetes commands also work on other Linux systems.

> **What you will build:** Minikube -> Kubernetes MCP Server -> VS Code -> LLM -> nginx pod and service.

## 1. Prerequisites

- Ubuntu WSL
- At least 2 CPUs, 2 GB memory, and 20 GB free disk space
- Docker Desktop with WSL integration, or another Minikube-supported container runtime
- VS Code with MCP support

Update Ubuntu and install basic tools:

```bash
sudo apt-get update
sudo apt-get install -y curl gpg ca-certificates conntrack
```

Install `kubectl` if it is not already available:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

The command above targets `amd64`. On ARM64, replace `amd64` with `arm64`.

## 2. Install Minikube

The Docker driver is a good choice for Ubuntu WSL when Docker Desktop is available:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
minikube version
```

Confirm that Docker is reachable from WSL:

```bash
docker version
```

If Docker is unavailable, start Docker Desktop and enable WSL integration for this Ubuntu distribution.

## 3. Start a minimal cluster without addons

Start a fresh, intentionally small cluster. `--addons=[]` keeps optional Minikube addons disabled:

```bash
minikube delete
minikube start --driver=docker --cpus=2 --memory=2048mb --addons=[]
kubectl config use-context minikube
```

Check the cluster:

Check the cluster context:

```bash
kubectl config current-context
minikube status
```

Expected context:

```text
minikube
```

## 4. Test Minikube

Run a temporary pod to verify scheduling, image pulling, DNS, and basic container execution:

```bash
kubectl run toolbox --image=busybox:1.36 --restart=Never --rm -it -- \
  sh -c 'echo pod-ok; nslookup kubernetes.default.svc.cluster.local'
kubectl get nodes -o wide
kubectl cluster-info
```

If the test fails, inspect the cluster before continuing:

```bash
minikube status
kubectl get events --all-namespaces --sort-by=.lastTimestamp
minikube logs
```

## 5. Install Helm

The apt repository install can fail if WSL cannot validate the Helm repo certificate chain. The release tarball method is a reliable fallback.

```bash
ARCH=$(dpkg --print-architecture)
case "$ARCH" in
  amd64) HELM_ARCH=amd64 ;;
  arm64) HELM_ARCH=arm64 ;;
  *) echo "Unsupported architecture: $ARCH"; exit 1 ;;
esac

cd /tmp
curl -fsSLO "https://get.helm.sh/helm-v3.16.3-linux-${HELM_ARCH}.tar.gz"
curl -fsSLO "https://get.helm.sh/helm-v3.16.3-linux-${HELM_ARCH}.tar.gz.sha256sum"
sha256sum -c "helm-v3.16.3-linux-${HELM_ARCH}.tar.gz.sha256sum"
tar -xzf "helm-v3.16.3-linux-${HELM_ARCH}.tar.gz"
sudo install -m 0755 "linux-${HELM_ARCH}/helm" /usr/local/bin/helm
helm version
```

## 6. Install Kubernetes MCP Server with Helm

Add the HelmForge chart repository and install the Kubernetes MCP Server into its own namespace:

```bash
helm repo add helmforge https://repo.helmforge.dev
helm repo update helmforge

helm upgrade --install kubernetes-mcp-server helmforge/kubernetes-mcp-server \
  --namespace kubernetes-mcp-server \
  --create-namespace \
  --wait \
  --timeout 5m
```

Verify the release:

```bash
helm status kubernetes-mcp-server --namespace kubernetes-mcp-server
kubectl get pods -n kubernetes-mcp-server
kubectl get svc,endpoints -n kubernetes-mcp-server
```

The chart defaults to a safer read-only profile:

- `readOnly: true`
- `disableDestructive: true`
- `clusterProvider: in-cluster`
- ServiceAccount bound to the Kubernetes `view` ClusterRole

Helm installs the server into its own namespace and manages the Kubernetes resources as one release. Read-only and destructive-operation safeguards are useful defaults. The repair exercise later requires write access, so only enable the chart's documented write settings in this disposable local cluster.

## 7. Configure VS Code for MCP

Forward the MCP service to localhost:

```bash
kubectl port-forward svc/kubernetes-mcp-server-kubernetes-mcp-server 8080:8080 \
  -n kubernetes-mcp-server
```

Create `.vscode/mcp.json` in the workspace:

```json
{
  "servers": {
    "minikube-kubernetes-mcp": {
      "type": "http",
      "url": "http://127.0.0.1:8080/mcp"
    }
  }
}
```

Validate the local connection:

```bash
curl -i http://127.0.0.1:8080/healthz
curl -i -H 'Accept: application/json, text/event-stream' http://127.0.0.1:8080/mcp
```

`/healthz` should return `200 OK`. A plain `GET /mcp` may return `405 Method Not Allowed`, which is expected because the MCP endpoint expects client MCP traffic over `POST`.

In VS Code, reload the window or use the Command Palette command for listing MCP servers.

Keep the port-forward terminal running while using the MCP server.

## 8. Use MCP prompts to create nginx

Paste these prompts into VS Code chat after the MCP server is connected.

### Create the pod

```text
Using the Kubernetes MCP server, create a pod named nginx in the default namespace with image nginx:latest. Do not create a Deployment. Wait until it is Ready, then show its status, node, and container image.
```

Equivalent manual command:

```bash
kubectl run nginx --image=nginx:latest --restart=Never --namespace=default
kubectl wait --for=condition=Ready pod/nginx --namespace=default --timeout=120s
kubectl get pod nginx --namespace=default -o wide
```

### Create and test the service

```text
Expose the nginx pod in the default namespace as a ClusterIP service named nginx on port 80, targeting container port 80. Show the service selector and endpoints, and tell me whether the service has a ready backend.
```

Equivalent manual commands:

```bash
kubectl expose pod nginx --name=nginx --type=ClusterIP --port=80 --target-port=80 --namespace=default
kubectl get svc nginx --namespace=default -o wide
kubectl get endpoints nginx --namespace=default
```

Forward and test the service:

```bash
kubectl port-forward svc/nginx 8081:80 --namespace=default
curl -I http://127.0.0.1:8081/
```

Expected response includes:

```text
HTTP/1.1 200 OK
Server: nginx
```

## 9. Deliberately make nginx faulty

Keep the service, but recreate the one-off pod with an image tag that does not exist:

```bash
kubectl delete pod nginx --namespace=default
kubectl run nginx --image=nginx:latestsscsdc --restart=Never --namespace=default
kubectl get pod nginx --namespace=default -w
```

Press `Ctrl+C` after the status changes to `ErrImagePull` or `ImagePullBackOff`. The service should now have no ready backend.

Collect evidence without guessing:

```bash
kubectl get pod nginx --namespace=default -o wide
kubectl describe pod nginx --namespace=default
kubectl logs pod/nginx --namespace=default
kubectl get events --namespace=default --sort-by=.lastTimestamp
kubectl get endpoints nginx --namespace=default
```

## 10. Ask the LLM to solve it using MCP

Paste this prompt into VS Code:

```text
Use the Kubernetes MCP server to troubleshoot the nginx workload in the default namespace.

1. Inspect the nginx pod, events, container status, logs, and nginx service endpoints.
2. Identify the root cause from observed evidence; do not guess.
3. Explain the smallest safe repair.
4. If write operations are enabled for this local lab, repair the workload so nginx:latest is running and the nginx service has a ready endpoint. Otherwise, give me the exact kubectl command.
5. Verify the repair by checking pod readiness, service endpoints, and an HTTP response through the service.

Do not delete unrelated resources. Ask for confirmation before destructive operations.
```

The expected diagnosis is that `nginx:latestsscsdc` is an invalid image reference. A valid repair for this one-off pod is:

```bash
kubectl delete pod nginx --namespace=default
kubectl run nginx --image=nginx:latest --restart=Never --namespace=default
kubectl wait --for=condition=Ready pod/nginx --namespace=default --timeout=120s
kubectl get endpoints nginx --namespace=default
curl -I http://127.0.0.1:8081/
```

If MCP reports that it cannot change the pod, the server is behaving as configured: the chart defaults to read-only access and destructive operations disabled. Review the chart values and use write access only for this local lab, never as an automatic production default.

## 11. Run nginx manually

A one-off pod can be created with:

A one-off pod can be created with:

```bash
kubectl run nginx --image=nginx:latest --restart=Never --namespace=default
kubectl wait --for=condition=Ready pod/nginx --namespace=default --timeout=120s
kubectl get pod nginx --namespace=default -o wide
```

Expose it with a ClusterIP service:

```bash
kubectl expose pod nginx --name=nginx --type=ClusterIP --port=80 --target-port=80 --namespace=default
kubectl get svc nginx --namespace=default -o wide
kubectl get endpoints nginx --namespace=default
```

Forward the service to localhost:

```bash
kubectl port-forward svc/nginx 8081:80 --namespace=default
```

Test it:

```bash
curl -I http://127.0.0.1:8081/
```

Expected response includes:

```text
HTTP/1.1 200 OK
Server: nginx
```

## 12. Troubleshooting nginx

During the session, the `nginx` pod stopped running for two reasons:

1. It was created as a one-off pod with `restartPolicy: Never`, so once the container exited successfully the pod became `Completed`.
2. The image field was later changed to an invalid tag such as `nginx:latests` or `nginx:latestsscsdc`, which caused image pull failures.

Check pod state, events, and logs:

```bash
kubectl get pods --all-namespaces
kubectl describe pod nginx --namespace=default
kubectl logs pod/nginx --namespace=default
kubectl get events --namespace=default --sort-by=.lastTimestamp
```

For a long-running nginx workload, use a Deployment instead of a one-off pod:

```bash
kubectl delete pod nginx --namespace=default --ignore-not-found
kubectl create deployment nginx --image=nginx:latest --namespace=default
kubectl patch deployment nginx --namespace=default --type=merge \
  -p '{"spec":{"template":{"metadata":{"labels":{"run":"nginx"}}}}}'
kubectl rollout status deployment/nginx --namespace=default --timeout=120s
kubectl get pods --namespace=default -l run=nginx -o wide
kubectl get svc,endpoints --namespace=default nginx -o wide
```

If an old port-forward fails with `container not running`, restart the port-forward:

```bash
kubectl port-forward svc/nginx 8081:80 --namespace=default
```

## 13. Useful Cleanup Commands

Remove the nginx test workload:

```bash
kubectl delete service nginx --namespace=default --ignore-not-found
kubectl delete deployment nginx --namespace=default --ignore-not-found
kubectl delete pod nginx --namespace=default --ignore-not-found
```

Remove the MCP server:

```bash
helm uninstall kubernetes-mcp-server --namespace kubernetes-mcp-server
kubectl delete namespace kubernetes-mcp-server
```

Delete the entire local cluster when the lab is complete:

```bash
minikube delete
```
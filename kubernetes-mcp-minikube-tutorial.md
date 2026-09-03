# Kubernetes MCP Server on Minikube

This tutorial documents the steps from this chat session: installing Helm on Ubuntu WSL, deploying the Kubernetes MCP Server to Minikube with Helm, connecting it to VS Code, and testing the cluster with an nginx workload.

## Prerequisites

- Ubuntu WSL
- Minikube running
- `kubectl` configured for the `minikube` context
- `curl`, `gpg`, and `sha256sum`

Check the cluster context:

```bash
kubectl config current-context
minikube status
```

Expected context:

```text
minikube
```

## Install Helm

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

## Install Kubernetes MCP Server

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

## Connect VS Code to the MCP Server

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

## Run nginx on Minikube

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

## Troubleshooting nginx

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

## Useful Cleanup Commands

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
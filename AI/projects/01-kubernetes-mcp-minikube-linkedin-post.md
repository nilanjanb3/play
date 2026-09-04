# LinkedIn Post

I wanted to understand what Kubernetes MCP looks like in a real troubleshooting workflow, so I built a small local lab with Minikube, Helm, VS Code, and an nginx workload.

The setup is simple:

- A minimal Minikube cluster running without addons
- Kubernetes MCP Server installed with Helm
- VS Code connected to the MCP endpoint
- An nginx pod exposed through a ClusterIP service

Then I introduced a realistic failure: I recreated the pod with an invalid image tag, `nginx:latestsscsdc`.

Instead of guessing, I asked the LLM to use MCP to:

1. Inspect the pod status, events, container state, logs, and service endpoints
2. Identify the root cause from the evidence
3. Explain the smallest safe repair
4. Restore the workload and verify the service response

The diagnosis was straightforward once the right evidence was available: Kubernetes could not pull the image, so the pod entered `ImagePullBackOff` and the service had no ready backend.

What I found most useful was the workflow itself. MCP gave the LLM a controlled way to inspect the cluster, while the prompt kept the investigation grounded in observable state instead of assumptions. The chart's read-only defaults also made a good distinction between diagnosis and repair: write access should be deliberate, especially outside a disposable local lab.

I documented the complete setup, commands, prompts, troubleshooting steps, and cleanup instructions here:

[GitHub project link]

This was a small experiment, but it made the Kubernetes + MCP workflow much easier to reason about. The next step would be testing the same pattern with a Deployment, rollout failure, or a misconfigured service selector.

#Kubernetes #MCP #Minikube #DevOps #CloudNative

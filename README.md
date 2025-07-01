# Local Gateway API Demo on macOS with Colima

This repository contains example manifests and instructions for testing the Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) on macOS using [Colima](https://github.com/abiosoft/colima).

## Prerequisites

- macOS with [Homebrew](https://brew.sh/) installed
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/) command-line tool
- [`colima`](https://github.com/abiosoft/colima) for running Kubernetes locally
- [`istioctl`](https://istio.io/latest/docs/setup/install/istioctl/) to install the Istio controller

Install the tools via Homebrew:

```bash
brew install kubectl colima istioctl
```

## Start Colima with Kubernetes

Create a local cluster with enough resources:

```bash
colima start --cpu 2 --memory 4 --disk 20 --kubernetes
```

Colima will create a Kubernetes context named `colima`. Verify it:

```bash
kubectl config current-context
kubectl get nodes
```

## Install Gateway API CRDs

Apply the official Gateway API CRDs to the cluster:

```bash
kubectl apply -k "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.0.0"
```

## Install Istio minimal profile

The manifests in this repository use Istio as the Gateway API controller. Install it with `istioctl`:

```bash
istioctl install --set profile=minimal -y
```

## Deploy the example

Apply all manifests from the `manifests` directory:

```bash
kubectl apply -f manifests/
```

This creates:

- An `istio` `GatewayClass`
- A `demo-gateway` `Gateway` in the `infra` namespace
- Two application namespaces (`app1` and `app2`) labeled so they can attach HTTPRoutes
- Echo deployments and services in each namespace
- Example `HTTPRoute` objects `echo.example.com` and `echo2.example.com`
- RBAC Roles and bindings that only let each team manage resources in its own namespace

The included `rbac.yaml` manifest defines Roles and RoleBindings limiting each
team to its own namespace. Team A operates in `app1` using the `dev-team-a`
ServiceAccount, while Team B uses `dev-team-b` in `app2`. Neither service
account has permissions in the other team's namespace or the `infra` namespace.

Wait for the gateway to be ready:

```bash
kubectl get gateway -n infra
```

## Access the application

Find the external address of the gateway service (provided by Istio):

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Map the address to each example host and curl it (replace `ADDRESS` with the `EXTERNAL-IP` or `CLUSTER-IP` you get above):

```bash
# app1
curl -H "Host: echo.example.com" http://ADDRESS/
# app2
curl -H "Host: echo2.example.com" http://ADDRESS/
```

You should see `hello from echo-app` and `hello from echo-app-2` in the respective responses.

## Cleanup

To stop the cluster:

```bash
colima stop
```

You can delete it entirely with `colima delete` if desired.

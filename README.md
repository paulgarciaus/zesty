# Kubernetes Debugging & Deployment Assignment

## Scenario

You have joined a team that maintains a Kubernetes-based platform. A local development cluster based on **[Kind](https://kind.sigs.k8s.io)** is used for day-to-day work. The manifests and Helm charts you received are not in a perfect state – they contain intentional issues that you must debug and fix.

Your goal is to:

- Create and verify a local Kubernetes cluster using **kind**.
- Install **cert-manager** using Helm.
- Deploy a simple **nginx** application using a provided (and intentionally imperfect) Helm chart.
- Expose the application over HTTPS using an Ingress and cert-manager.
- Debug and fix issues until the setup works end-to-end.

You are expected to work with the existing configuration instead of throwing it away and rebuilding everything from scratch.

---

## Prerequisites

- Docker (or another container runtime compatible with kind).
- `kind` (Kubernetes in Docker).
- `kubectl`.
- `helm` v3.
- Unix-like environment (macOS or Linux is recommended).
- Internet access to pull container images and Helm charts.

---

## Repository Layout

- `README.md`
  - This file, with candidate instructions.
- `cert-manager/`
  - `cluster-issuer.yaml` – ClusterIssuer manifest for cert-manager.
  - `certificate.yaml` – Certificate manifest for the nginx application.
- `charts/nginx-app/`
  - A Helm chart for the sample nginx application (intentionally contains some issues).

You may add additional files (scripts, notes, manifests) if they help you complete the assignment.

---

## High-Level Tasks

1. **Create a local Kind cluster**.
2. **Install cert-manager** using Helm and the provided values file.
3. **Create a ClusterIssuer and Certificate** for the host `nginx.local`.
4. **Deploy the nginx application** using the provided Helm chart.
5. **Expose nginx over HTTPS** using an Ingress resource and cert-manager-issued certificate.
6. **Debug and fix** any issues you encounter so that the full flow works.
7. **Document** what was broken and how you fixed it.

The manifests and chart in this repository are intentionally not perfect. You should expect that some of them will fail or behave incorrectly until you fix them.

---

## 1. Create the kind Cluster

Use [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) to create a local cluster. Verify basic cluster health.

---

## 2. Install cert-manager

Add the Jetstack Helm repository and install cert-manager using the provided values file:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

Wait for the cert-manager pods to become ready:

```bash
kubectl get pods -n cert-manager
```

Then apply the ClusterIssuer and Certificate manifests:

```bash
kubectl apply -f cert-manager/cluster-issuer.yaml
kubectl apply -f cert-manager/certificate.yaml
```

---

## 3. Deploy the nginx Application via Helm

Deploy the sample application to a dedicated namespace using the provided Helm chart:

```bash
helm install nginx-app ./charts/nginx-app \
  --namespace nginx \
  --create-namespace
```

Inspect the resulting resources.

You may edit any of the manifests or Helm templates in this repository.
Prefer making **targeted changes** that fix the root cause rather than rewriting everything from scratch.

---

## 4. Accessing the nginx Application

The goal is to access the nginx application from your local machine using port-forwarding.

1. Ensure there is an **Ingress controller** installed in the cluster (for example, `ingress-nginx`).
   You may install and configure an ingress controller of your choice; this repository does not include one.

2. Once the nginx application is deployed and all issues are resolved, use `kubectl port-forward` to access the service.

3. **Provide the exact `curl` command** you used to successfully make an HTTP request to the nginx application through the port-forward.
   Include this command in your final report.

---

## 5. What to Submit

Please provide:

1. **Your code changes**
   - A Git repository or archive containing this assignment, including any modifications you made to manifests, values, or templates.
   - You may add helper scripts or documentation if they clarify your approach.

2. **A short written report (1–2 pages maximum)**
   - What was broken.
   - How you diagnosed each issue.
   - The root cause for each issue.
   - The fix you applied.
   - The tools and commands that helped you most.
   - What you would change in a real environment to prevent similar issues from recurring.

---

## Expectations

- You can use any tools you find helpful, including AI tools.
- If needed, edit the provided YAML and Helm templates.
- Please avoid completely replacing everything with a brand new setup; we want to see how you debug and fix an existing system.
- Commit your changes to a local Git repository.
- Document your findings in a report.

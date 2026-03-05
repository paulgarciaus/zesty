# Kubernetes Debugging & Deployment - Report

## Summary

Successfully debugged and deployed the nginx application on a local Kind cluster with HTTPS access via cert-manager and ingress-nginx.

---

## Issues Found and Fixed

### 1. ClusterIssuer - Deprecated API Version

**File:** `cert-manager/cluster-issuer.yaml`

**Issue:** Used deprecated API version `cert-manager.io/v1alpha2`

**Root Cause:** The API version `v1alpha2` was deprecated in cert-manager v1.0+ and removed in newer versions.

**Fix:** Changed to stable API version:
```yaml
apiVersion: cert-manager.io/v1
```

---

### 2. Values.yaml - Invalid Port Configuration

**File:** `charts/nginx-app/values.yaml`

**Issues:**
- `service.port` was set to `"eighty"` (string) instead of `80` (integer)
- `service.targetPort` was `8080` but nginx container listens on `80`
- `ingress.tls.secretName` was `nginx-app-tls` but certificate creates `nginx-tls`

**Root Cause:** 
- Typo/mistake in port value (string instead of integer)
- Mismatch between container port and targetPort
- Inconsistent naming between certificate and ingress TLS secret

**Fix:**
```yaml
service:
  type: ClusterIP
  port: 80
  targetPort: 80

ingress:
  tls:
    secretName: nginx-tls
```

---

### 3. Deployment.yaml - Incorrect Indentation

**File:** `charts/nginx-app/templates/deployment.yaml`

**Issue:** The `ports:` block was at the wrong indentation level (sibling to `containers` instead of nested under the container)

**Root Cause:** YAML indentation error - `ports` was not properly nested under the container spec

**Fix:** Corrected indentation to nest `ports` under the container:
```yaml
containers:
  - name: nginx
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    imagePullPolicy: {{ .Values.image.pullPolicy }}
    ports:
      - containerPort: 80
        name: http
```

---

### 4. Service.yaml - Selector Mismatch

**File:** `charts/nginx-app/templates/service.yaml`

**Issue:** Selector used `{{ .Release.Name }}-prod` but deployment pods are labeled with `{{ .Release.Name }}`

**Root Cause:** Incorrect label selector that doesn't match the pod labels from the deployment

**Fix:**
```yaml
selector:
  app: {{ .Release.Name }}
```

---

### 5. Ingress.yaml - Deprecated API and Format

**File:** `charts/nginx-app/templates/ingress.yaml`

**Issues:**
- Used deprecated API `networking.k8s.io/v1beta1`
- Used deprecated annotation `kubernetes.io/ingress.class`
- Used old backend format (`serviceName`/`servicePort`) instead of service reference

**Root Cause:** Ingress API was promoted to `v1` in Kubernetes 1.19+, requiring:
- `ingressClassName` field instead of annotation
- Service reference format for backend

**Fix:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-cluster-issuer
spec:
  ingressClassName: {{ .Values.ingress.className }}
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Values.ingress.tls.secretName }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-service
                port:
                  number: {{ .Values.service.port }}
```

---

## Tools and Commands Used

### Diagnosis Commands
```bash
# Check YAML syntax and structure
cat charts/nginx-app/templates/deployment.yaml
cat charts/nginx-app/templates/service.yaml

# Verify API versions
kubectl api-resources | grep -i ingress
kubectl api-resources | grep -i clusterissuer

# Check resource status
kubectl get all -n nginx
kubectl get ingress -n nginx
kubectl get certificate -n nginx
kubectl describe pod <pod-name> -n nginx
```

### Deployment Commands
```bash
# Create cluster
kind create cluster --name nginx-cluster --config kind-config.yaml

# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Install ingress-nginx
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Apply cert-manager resources
kubectl apply -f cert-manager/cluster-issuer.yaml
kubectl apply -f cert-manager/certificate.yaml

# Deploy nginx app
helm install nginx-app ./charts/nginx-app --namespace nginx
```

---

## Successful HTTPS Access Command

```bash
curl -k --resolve nginx.local:443:127.0.0.1 https://nginx.local/
```

Output:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
</body>
</html>
```

---

## Recommendations for Production

1. **API Version Management**: Always use stable API versions (`v1` instead of `alpha`/`beta`). Use `kubectl api-resources` to check available versions.

2. **Linting**: Implement YAML linting and Helm chart testing (`helm lint`, `helm template`) in CI/CD pipelines to catch syntax errors early.

3. **Label Consistency**: Use a consistent labeling strategy across all resources. Consider using Helm hooks or helpers to ensure selectors match.

4. **Port Validation**: Use integer values for ports in Kubernetes manifests. Consider using `values.schema.json` to enforce types.

5. **Secret Name Coordination**: Ensure TLS secret names are consistent between Certificate and Ingress resources. Use a single source of truth.

6. **Pre-deployment Validation**: Use `kubectl apply --dry-run=client` and `helm template` to validate manifests before applying.

---

## Files Modified

| File | Change |
|------|--------|
| `cert-manager/cluster-issuer.yaml` | API version `v1alpha2` → `v1` |
| `charts/nginx-app/values.yaml` | Port `"eighty"` → `80`, targetPort `8080` → `80`, secretName fixed |
| `charts/nginx-app/templates/deployment.yaml` | Fixed `ports` indentation |
| `charts/nginx-app/templates/service.yaml` | Fixed selector label |
| `charts/nginx-app/templates/ingress.yaml` | API `v1`, new backend format, `ingressClassName` |

## Files Created

| File | Purpose |
|------|---------|
| `kind-config.yaml` | Kind cluster configuration with port mappings |
| `REPORT.md` | This report documenting all issues and fixes |

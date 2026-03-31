# Migration Guide: Ingress NGINX → Traefik

**Date**: March 2026
**Reason**: [Ingress NGINX has been retired](https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/) as of March 2026. No further releases, bugfixes, or security patches will be provided.

---

## Table of Contents

1. [Background](#background)
2. [What Changed in This Project](#what-changed-in-this-project)
3. [Prerequisites](#prerequisites)
4. [Migration Steps](#migration-steps)
5. [Rollback Plan](#rollback-plan)
6. [Applying to Other Projects](#applying-to-other-projects)
7. [FAQ](#faq)

---

## Background

Kubernetes SIG Network and the Security Response Committee announced the retirement of
Ingress NGINX in November 2025. As of March 2026, the project receives no further
maintenance, security patches, or releases. The GitHub repositories are now read-only.

**Traefik** is a widely adopted, actively maintained ingress controller that natively
supports the standard Kubernetes Ingress API (`networking.k8s.io/v1`). This means the
migration from Ingress NGINX to Traefik is straightforward — the Ingress resource
specification remains the same; only the `ingressClassName` and any controller-specific
annotations need to change.

Key Traefik documentation:
- [Traefik Kubernetes Ingress Provider](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [Traefik Kubernetes Ingress Routing](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/)

---

## What Changed in This Project

| File | Change |
|------|--------|
| `charts/ssi-dim-wallet-stub/values.yaml` | `wallet.ingress.className`: `nginx` → `traefik` |
| `charts/ssi-dim-wallet-stub-memory/values.yaml` | `wallet.ingress.className`: `nginx` → `traefik` |
| `charts/ssi-dim-wallet-stub/README.md` | Updated default value in docs table |
| `charts/ssi-dim-wallet-stub-memory/README.md` | Updated default value in docs table |

**No changes were needed to the Ingress templates** (`templates/ingress.yaml`). The
templates already use the standard `networking.k8s.io/v1` Ingress API with:
- `ingressClassName` (read from values)
- `pathType: Prefix`
- Standard host-based routing
- Standard TLS with secrets

All of these are natively supported by Traefik's Kubernetes Ingress provider.

---

## Prerequisites

### 1. Install Traefik in Your Cluster

If Traefik is not already installed:

```bash
# Using Helm
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik --namespace traefik --create-namespace
```

### 2. Verify the Traefik IngressClass Exists

```bash
kubectl get ingressclass
```

Expected output should include:
```
NAME      CONTROLLER                      PARAMETERS   AGE
traefik   traefik.io/ingress-controller   <none>       5m
```

### 3. (Optional) Remove Ingress NGINX

After verifying Traefik works:
```bash
# If installed via Helm
helm uninstall ingress-nginx --namespace ingress-nginx
kubectl delete namespace ingress-nginx
```

---

## Migration Steps

### Step 1: Update `values.yaml`

Change the ingress class from `nginx` to `traefik`:

```yaml
# Before
wallet:
  ingress:
    className: nginx

# After
wallet:
  ingress:
    className: traefik
```

### Step 2: Replace Any NGINX-Specific Annotations

If your deployment uses nginx-specific annotations, replace them with Traefik equivalents:

| NGINX Annotation | Traefik Equivalent |
|---|---|
| `nginx.ingress.kubernetes.io/rewrite-target: /` | Use `traefik.ingress.kubernetes.io/router.middlewares` with a StripPrefix middleware |
| `nginx.ingress.kubernetes.io/ssl-redirect: "true"` | `traefik.ingress.kubernetes.io/router.tls: "true"` or use entrypoint redirect |
| `nginx.ingress.kubernetes.io/proxy-body-size: "10m"` | `traefik.ingress.kubernetes.io/router.middlewares` with Buffering middleware |
| `nginx.ingress.kubernetes.io/proxy-read-timeout: "60"` | Configure via ServersTransport |
| `nginx.ingress.kubernetes.io/cors-*` | `traefik.ingress.kubernetes.io/router.middlewares` with Headers middleware |
| `nginx.ingress.kubernetes.io/whitelist-source-range` | `traefik.ingress.kubernetes.io/router.middlewares` with IPAllowList middleware |
| `nginx.ingress.kubernetes.io/rate-limit-*` | `traefik.ingress.kubernetes.io/router.middlewares` with RateLimit middleware |

> **Note**: This project uses `annotations: {}` by default, so no annotation changes were
> needed. The table above is provided for reference when migrating other projects.

### Step 3: Upgrade the Helm Release

```bash
helm upgrade <release-name> charts/ssi-dim-wallet-stub \
  --namespace wallet \
  --set wallet.ingress.enabled=true \
  --set wallet.host=<your-hostname>
```

### Step 4: Verify the Ingress

```bash
# Check the ingress was created and assigned
kubectl get ingress -n wallet

# Verify Traefik picked it up
kubectl describe ingress ssi-dim-wallet-ingress -n wallet

# Test connectivity
curl -v https://<your-hostname>/
```

---

## Rollback Plan

If you need to revert to NGINX temporarily (while it's still operational in your cluster):

```bash
helm upgrade <release-name> charts/ssi-dim-wallet-stub \
  --namespace wallet \
  --set wallet.ingress.className=nginx
```

> **Warning**: Ingress NGINX is no longer maintained. Only use this as a short-term
> rollback while troubleshooting Traefik configuration.

---

## Applying to Other Projects

This guide can be used as a template for migrating any Kubernetes project from
Ingress NGINX to Traefik. Here's the general checklist:

### General Migration Checklist

- [ ] **Verify Traefik is installed** in the target cluster and the `traefik` IngressClass exists
- [ ] **Search for `nginx` references** in all Helm values, templates, and documentation:
  ```bash
  grep -rn "nginx" charts/ --include="*.yaml" --include="*.yml" --include="*.md"
  ```
- [ ] **Change `ingressClassName`**: Replace `nginx` with `traefik` in all `values.yaml` files
- [ ] **Audit annotations**: Identify any `nginx.ingress.kubernetes.io/*` annotations and replace with Traefik equivalents (see annotation mapping table above)
- [ ] **Check `pathType`**: Traefik supports `Exact` and `Prefix` — the same as NGINX. No changes needed.
- [ ] **Verify TLS configuration**: Traefik uses the same standard `tls` block with secrets. If you use cert-manager, it continues to work unchanged.
- [ ] **Test with `helm template`**: Render templates locally to verify correctness:
  ```bash
  helm template test charts/<your-chart> --set ingress.enabled=true | grep -A 30 "kind: Ingress"
  ```
- [ ] **Deploy to a staging environment first**
- [ ] **Update documentation**: Change default values in README tables and any deployment guides
- [ ] **Verify connectivity**: Test all endpoints through the new ingress

### Traefik-Specific Features to Consider

If you want to go beyond basic Ingress compatibility, Traefik also supports:

- **IngressRoute CRD**: Traefik's native CRD for more advanced routing (optional, not required)
- **Middleware annotations**: Rate limiting, circuit breakers, headers, etc. directly via annotations
- **Dashboard**: Built-in dashboard for monitoring routes (`traefik.ingress.kubernetes.io/router.entrypoints`)
- **Gateway API**: Traefik also supports the newer Kubernetes Gateway API as a future-proof alternative

---

## FAQ

### Q: Do I need to change my Ingress template YAML?
**A**: No, if your templates use the standard `networking.k8s.io/v1` Ingress API. Traefik is
a drop-in replacement for the Ingress controller — it reads the same Ingress resources.
The only change is the `ingressClassName` value.

### Q: What about `pathType`?
**A**: Traefik supports both `Exact` and `Prefix` pathTypes, just like NGINX. `Exact` maps
to Traefik's `Path` matcher, and `Prefix` maps to `PathPrefix`.

### Q: Will my TLS certificates still work?
**A**: Yes. Traefik reads the same `tls.secretName` from the Ingress spec. If you use
cert-manager, it continues to work without changes.

### Q: What if I have NGINX snippets?
**A**: NGINX snippets (`nginx.ingress.kubernetes.io/configuration-snippet` or
`server-snippet`) have no Traefik equivalent. You'll need to implement the same logic
using Traefik middlewares (via annotations or IngressRoute CRDs).

### Q: Can I run both NGINX and Traefik side by side during migration?
**A**: Yes. Each Ingress resource specifies its `ingressClassName`. Resources with
`className: nginx` will be handled by NGINX, and those with `className: traefik` will
be handled by Traefik. This allows gradual migration.

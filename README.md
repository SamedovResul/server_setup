# K3s + NGINX Ingress + cert-manager Setup

End-to-end guide for standing up a single-node k3s cluster with NGINX
Ingress and automatic Let's Encrypt TLS via cert-manager. Example domain
used throughout: `dev.entesk.com` — swap for your own.

## Prerequisites

- A VM/droplet with root access and a public IPv4 address
- Ports `80`, `443`, and `6443` open in any cloud firewall (not just security groups on your own IP — Let's Encrypt's HTTP-01 challenge needs to be reachable from the whole internet)
- A domain you control, with access to its registrar and DNS provider

---

## 1. Install k3s (disable built-in Traefik)

```bash
curl -sfL https://get.k3s.io \
  | INSTALL_K3S_EXEC="--disable=traefik --tls-san dev.entesk.com --tls-san $(curl -4 -s ifconfig.me)" sh -
```

- `--disable=traefik` — you're running your own NGINX ingress instead of k3s's bundled one, so there's no port 80/443 conflict.
- `--tls-san` — SANs for the k3s **API server's own cert** (port 6443, what `kubectl` talks to). This is unrelated to the Let's Encrypt cert for your app. Repeat the flag once per value — it does **not** accept a comma-separated list in one flag.

Verify:
```bash
kubectl get nodes
```

---

## 2. Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx
```

Wait until the controller pod is `Running`.

---

## 3. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.3/cert-manager.yaml
kubectl get pods -n cert-manager
```

Wait until all three pods (`cert-manager`, `cainjector`, `webhook`) are `Running`.

---

## 4. DNS / Domain Setup — do this before requesting any certificate

This is the step most likely to break issuance, and it happens outside Kubernetes entirely.

1. **Create one DNS zone for the exact domain you're issuing a cert for** (e.g. `dev.entesk.com`) with your DNS provider (DigitalOcean, etc.).
   - Don't create a zone literally named `www.dev.entesk.com` as its own separate domain — add `www` as a record *inside* the zone for the real domain instead. Creating a hostname inside a zone that's already named `www.something` produces a broken `www.www.something`.
2. **Add an A record for the bare domain** (hostname `@`) pointing at your server's public IP. If you also want `www` to work, add a second A record for `www` in the same zone.
3. **At your registrar**, confirm the domain's nameservers are set to your DNS provider's nameservers (e.g. `ns1/2/3.digitalocean.com`). This is a separate step from anything inside the DNS provider's dashboard — it's what tells the internet "ask this provider about this domain" in the first place. Registrar-side nameserver changes can take anywhere from minutes to ~24 hours to propagate.
4. **Verify delegation actually resolves** before touching cert-manager:
   ```bash
   dig NS dev.entesk.com +short      # should return your DNS provider's nameservers
   dig A dev.entesk.com +short       # should match your server's public IP
   dig CAA dev.entesk.com +short     # empty is fine; SERVFAIL is not (see Troubleshooting)
   ```

---

## 5. Create a ClusterIssuer for Let's Encrypt

`clusterissuer.yaml`:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

```bash
kubectl apply -f clusterissuer.yaml
kubectl get clusterissuer
```

---

## 6. Deploy Your App (example)

`deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: your-image:tag
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f deployment.yaml
```

---

## 7. Create Ingress with TLS

`ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - dev.entesk.com
      secretName: my-app-tls
  rules:
    - host: dev.entesk.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
```

cert-manager watches the Ingress's `tls` block and automatically creates the matching `Certificate` resource — no separate `kubectl apply` needed for that part.

---

## 8. Verify the Certificate Issued

```bash
kubectl get certificate
kubectl describe certificate my-app-tls
```

Look for `Ready: True` in the conditions. That's success — HTTPS should now work at `https://dev.entesk.com`.

---

## Debugging

**General:**
```bash
kubectl describe certificate <name>
kubectl describe ingress <name>
kubectl logs -n cert-manager deploy/cert-manager
```

**If the Certificate's order is stuck `invalid`**, `describe certificate` alone often truncates the real reason. Go one level deeper:
```bash
kubectl get order -A
kubectl describe order <order-name>
kubectl get challenges -A
kubectl describe challenge <challenge-name>
```
The Challenge's `Reason:` field has the actual ACME failure — that's the message to read, not the generic "order is in invalid state" line on the Certificate.

**`DNS problem: SERVFAIL looking up CAA for <domain>`** — this is a DNS delegation problem, not a cert-manager problem. Let's Encrypt is required to check CAA at the exact domain *and* walk up to the parent apex, even for a subdomain-only certificate — so a broken apex zone breaks every certificate under it. Checklist:
1. `dig NS <apex-domain> +short` — empty means the registrar never delegated (or it hasn't propagated). Fix at the registrar.
2. `dig DS <apex-domain> +short` — if this returns anything, there's a stale DNSSEC key published at the registry that no longer matches your DNS provider's signing (or lack thereof). Remove/disable DNSSEC at the registrar.
3. Don't retry cert-manager in a loop while DNS is still propagating — you'll just collect more failed attempts without fixing anything, and burn into Let's Encrypt's rate limits.

**Forcing a fresh attempt** once the root cause is actually fixed:
```bash
kubectl delete certificaterequest <name>
kubectl get challenges -A
```

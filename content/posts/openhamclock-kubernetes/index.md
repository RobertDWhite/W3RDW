---
title: "OpenHamClock on Kubernetes"
date: "2026-02-10"
categories:
  - "hamclock"
  - "tutorials"
  - "kubernetes"
  - "ham"
  - "radio"
tags:
  - "hamclock"
  - "openhamclock"
  - "kubernetes"
  - "tutorials"
  - "ham"
  - "radio"
cover:
  image: "/posts/openhamclock/openhamclock.png"
  alt: "openhamclock"
  caption: "<text>"
  relative: true
aliases:
  - /posts/openhamclock-kubernetes/openhamclock/
  - /2026/openhamclock/
---

# Running OpenHamClock on Kubernetes (RKE2 and ArgoCD)

In this tutorial, I will walk through my Kubernetes version of running OpenHamClock in my homelab. In this
setup, everything is defined in YAML, committed to Git, and synced to the
cluster by ArgoCD. Access is handled by NGINX Ingress with TLS from
cert-manager, which is how I publish it at
[hamclock.w3rdw.radio](https://hamclock.w3rdw.radio).

If you want to follow my exact layout, use
[my RKE2 GitHub project](https://github.com/RobertDWhite/whitehouse-rke2/opehamclock). This
guide uses the `openhamclock/` directory.

### Workflow (high level):

1. Update OpenHamClock YAML in `openhamclock/`.
2. Commit and push to Git.
3. ArgoCD detects changes and syncs automatically.
4. Ingress terminates TLS and serves the app.

## Prerequisites

Software/cluster:

- RKE2 cluster (or any Kubernetes cluster)
- `ingress-nginx` installed
- `cert-manager` installed (for TLS certificates)
- ArgoCD installed and pointed at your Git repo
- A DNS record for your chosen hostname

## Step 1: Review the Kubernetes Stack Layout

The OpenHamClock stack lives here:

- [openhamclock/](https://github.com/RobertDWhite/whitehouse-rke2/tree/main/openhamclock)

Files in this stack:

- [openhamclock/00-namespace.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/00-namespace.yaml)
  Creates the `openhamclock` namespace.
- [openhamclock/10-pvc.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/10-pvc.yaml)
  Creates the `openhamclock-config` PVC.
- [openhamclock/20-deployment.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/20-deployment.yaml)
  Runs the `ggilman/openhamclock` container and mounts `/config`.
- [openhamclock/30-service.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/30-service.yaml)
  Exposes the app internally on port `3000`.
- [openhamclock/40-ingress.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/40-ingress.yaml)
  Publishes the app externally with TLS.
- [openhamclock/kustomization.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/kustomization.yaml)
  Ties everything together and controls image tag overrides.

## Step 2: Set Runtime Settings

Most customization happens in deployment/PVC/Kustomize:

Set timezone in:

- [openhamclock/20-deployment.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/20-deployment.yaml)

```yaml
env:
  - name: TZ
    value: "UTC"
```

Set persistent storage size in:

- [openhamclock/10-pvc.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/10-pvc.yaml)

```yaml
resources:
  requests:
    storage: 5Gi
```

Set image tag in:

- [openhamclock/kustomization.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/kustomization.yaml)

```yaml
images:
  - name: ggilman/openhamclock
    newTag: "1.2"
```

## Step 3: Configure Ingress (TLS + DNS)

Ingress is defined in:

- [openhamclock/40-ingress.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/openhamclock/40-ingress.yaml)

Current host and TLS secret:

```yaml
tls:
  - hosts:
      - hamclock.w3rdw.radio
    secretName: hamclock-w3rdw-radio-tls
rules:
  - host: hamclock.w3rdw.radio
```

This manifest uses:

- `ingressClassName: nginx`
- `cert-manager.io/cluster-issuer: letsencrypt-dns`
- `nginx.ingress.kubernetes.io/force-ssl-redirect: "true"`

Make sure your DNS record points to your ingress controller and your
`letsencrypt-dns` cluster issuer is working.

## Step 4: Deploy via ArgoCD

ArgoCD app definition:

- [argo-cd/applications/openhamclock-app.yaml](https://github.com/RobertDWhite/whitehouse-rke2/blob/main/argo-cd/applications/openhamclock-app.yaml)

It tracks `path: openhamclock` and auto-syncs:

```yaml
destination:
  namespace: openhamclock
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Commit your changes to Git and ArgoCD should deploy/update automatically.

## Step 5: Access OpenHamClock

Once synced, OpenHamClock should be available at:

- [https://hamclock.w3rdw.radio](https://hamclock.w3rdw.radio)

## Conclusion

That’s it! Congratulations! You now have OpenHamClock running as a fully declarative Kubernetes
app with persistent storage, TLS ingress, and GitOps deployment via ArgoCD.

This replaces ad-hoc container management with a repeatable workflow where all
changes are tracked in Git and continuously reconciled in-cluster.

> As always, if you have any questions or want to contribute to the above information, feel free to start a [Discussion on GitHub](https://github.com/RobertDWhite/W3RDW/discussions), [submit a GitHub](https://github.com/RobertDWhite/W3RDW/pulls) PR to recommend changes/fixes in the article, or reach out to me directly at [robert@w3rdw.radio](mailto:robert@w3rdw). Finally, feel free to [join my Matrix channel for W3RDW](https://matrix.to/#/%23w3rdw:white.fm) and chat with me there.
>
> Thanks for reading!
>
> 73 from Robert, W3RDW

---
title: "Exposing My Portfolio with Traefik Ingress on k3s"
date: 2026-10-31
draft: false
tags: ["k3s", "traefik", "kubernetes", "homelab", "networking"]
summary: "How I routed traffic to my portfolio on a 2-node k3s cluster using Traefik Ingress."
---

## Goal

Serve my Hugo portfolio from my k3s cluster at `portfolio.homelab.local`
instead of a raw IP and port.

**Traffic flow:** Browser → hosts file → node IP → Traefik → Ingress
rule → ClusterIP Service → one of 2 portfolio pods

## What I built

- **Deployment:** 2 replicas of my Hugo site, built into a Docker image
- **Service (ClusterIP):** one stable internal address for the pods
- **Ingress:** routes `portfolio.homelab.local` to that Service
- **Traefik:** the ingress controller bundled with k3s, which enforces the rule

## Steps I performed

### 1. Connect to the cluster node

```powershell
ssh <user>@<node-1-ip>
```

Opens a secure remote session on my first k3s node, where all the
`kubectl` commands below run.

### 2. Confirm Traefik is running

```bash
sudo kubectl get pods -n kube-system | grep traefik
```

Lists pods in the `kube-system` namespace (where k3s keeps its built-in
components) and filters for Traefik. `sudo` is required because the
cluster config file is readable only by root, which I kept that way on
purpose.

```
svclb-traefik-xxxxx   2/2   Running   4 (48d ago)   51d
svclb-traefik-xxxxx   2/2   Running   2 (48d ago)   51d
traefik-xxxxx         1/1   Running   1 (48d ago)   51d
```

Traefik is `Running`, with one `svclb` helper pod per node, so ports 80
and 443 are reachable on either node IP.

### 3. Confirm the Ingress rule exists

```bash
sudo kubectl get ingress
```

```
NAME        CLASS     HOSTS                     ADDRESS                     PORTS     AGE
portfolio   traefik   portfolio.homelab.local   <node-1-ip>,<node-2-ip>     80, 443   46d
```

The rule uses the `traefik` class, matches `portfolio.homelab.local`,
and lists both nodes as its address.

### 4. Review the Ingress configuration

```bash
sudo kubectl get ingress portfolio -o yaml
```

This prints the full manifest exactly as the cluster stores it. It sets
the ingress class to `traefik`, matches the host
`portfolio.homelab.local`, and routes all paths (`/`) to the portfolio
ClusterIP Service on port 80.

### 5. Map the hostname on my laptop

`.local` names don't exist in public DNS, so I added an entry to the
Windows hosts file and checked it:

```powershell
Get-Content C:\Windows\System32\drivers\etc\hosts
```

```
<node-ip>  portfolio.homelab.local
```

### 6. Test in the browser

Opening `http://portfolio.homelab.local` loads the portfolio, served by
one of the two pod replicas through Traefik.

## What I learned

A Service gives pods a stable internal address, while an Ingress is a
routing rule that Traefik acts on. k3s bundles Traefik, so no extra
install was needed. A hosts file entry was required because
`.local` names aren't in public DNS.

## What's next? 

Let see... Still thinking!


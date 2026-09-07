---
title: "Security Hardening Roadmap: Locking Down My Homelab"
date: 2026-09-07
draft: false
tags: ["security", "ssh", "tls", "wireguard", "vpn", "homelab"]
---

## Overview

With the k3s cluster and portfolio site running, I paused before adding more features to focus on securing what I'd already built. This post covers the hardening steps taken, why each mattered, and what's still on the roadmap for a more production-grade posture.

## Why Not Expose the Site Publicly

The original plan considered making the site reachable from the public internet. After weighing the risk, I decided against it:

- Port-forwarding a home router exposes the entire request path — router, Proxmox host, VM, Traefik, nginx — to internet-wide scanning and exploit attempts, not just the site's static content
- A static site has low content risk (no login, no database, no user input), but the underlying infrastructure risk remains regardless of how simple the content is
- The safer trade-off: keep the homelab private, and access it remotely through a VPN instead — same functional result, far smaller attack surface

## SSH Hardening: Key-Only Authentication

All three VMs (both k3s nodes and the WireGuard VM) were switched from password-based SSH to key-only authentication.

```bash
# Generated an ed25519 keypair locally
ssh-keygen -t ed25519

# Added the public key to each VM
mkdir -p ~/.ssh
echo "<public key>" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Then, on each VM:
```bash
sudo nano /etc/ssh/sshd_config
```
```
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```
```bash
sudo systemctl restart ssh
```

**Why this matters:** password auth is vulnerable to brute-force and credential-stuffing attacks. Key-based auth removes that attack vector entirely — an attacker would need the private key file itself, not just a guessed password.

**Lesson learned:** always test the new auth method in a second terminal session before closing the original — if the config is wrong, the still-open session is the only way back in without console access.

## TLS for the Ingress

The portfolio site was originally served over plain HTTP. I added a self-signed TLS certificate and enabled HTTPS through Traefik's Ingress.

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout portfolio.key -out portfolio.crt \
  -subj "/CN=portfolio.homelab.local" \
  -addext "subjectAltName=DNS:portfolio.homelab.local"
```

```bash
sudo k3s kubectl create secret tls portfolio-tls --cert=portfolio.crt --key=portfolio.key
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: portfolio
spec:
  tls:
  - hosts:
    - portfolio.homelab.local
    secretName: portfolio-tls
  rules:
  - host: portfolio.homelab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: portfolio
            port:
              number: 80
```

**Why self-signed instead of Let's Encrypt:** Let's Encrypt requires a publicly reachable domain to validate ownership. Since `portfolio.homelab.local` only exists on a private network, a self-signed cert is the appropriate (and free) choice here — the encryption itself is identical, only the browser trust chain differs.

**Interesting confirmation:** the TLS Secret and Ingress config only needed to be created once, on one control-plane node. Because the cluster uses k3s's embedded etcd for HA, both resources automatically replicated to the second node — verified by pointing a local DNS override at each node's IP in turn and confirming HTTPS worked identically on both.

## WireGuard VPN for Remote Access

Rather than exposing the site to the internet, I set up a dedicated WireGuard VPN server on its own VM, giving encrypted remote access to the entire home network without opening any inbound ports except the VPN itself.

```bash
sudo apt install wireguard -y
wg genkey | sudo tee privatekey | wg pubkey | sudo tee publickey
```

Server config (`/etc/wireguard/wg0.conf`):
```ini
[Interface]
PrivateKey = <server private key>
Address = 10.8.0.1/24
ListenPort = 51820
SaveConfig = true
```

IP forwarding and NAT were enabled so VPN clients can reach the wider LAN, not just the VPN server itself:
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
sudo iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
```

**Why WireGuard specifically:** it's fast, has a minimal attack surface (it doesn't respond to unauthenticated packets at all, making it invisible to port scanners), and is the modern standard for VPN tunneling — only one UDP port (51820) would ever need forwarding if remote access beyond the LAN were needed later.

**Verified working:** tested the tunnel from a phone on mobile data (a genuinely separate network, not just the same WiFi) and successfully reached the portfolio site over HTTPS through the tunnel — confirming real remote access, not just a same-LAN illusion.

## Current Security Posture Summary

| Area | Status |
|---|---|
| SSH access | Key-only, password auth disabled |
| Site transport | HTTPS (self-signed cert) |
| Remote access | WireGuard VPN, no public exposure |
| Cluster HA | Config/secrets replicate via embedded etcd |

## What's Still on the Roadmap

- **RBAC**: currently using unrestricted `sudo kubectl` access — no role separation yet
- **NetworkPolicies**: all pods can currently talk to all other pods; a default-deny policy with explicit allow rules would better isolate workloads
- **Encrypted Secrets at rest**: Kubernetes Secrets are base64-encoded by default, not encrypted — enabling encryption at rest would close this gap
- **Automated certificate rotation**: the self-signed cert is valid for 1 year and will need manual renewal; a tool like cert-manager would automate this in a real deployment

## What I Learned

- Security hardening is a series of deliberate trade-offs, not a single checklist — deciding *not* to expose the site publicly was as important a decision as any config change
- How Kubernetes' etcd-backed HA actually behaves in practice, not just in theory, when it comes to replicating Secrets and Ingress config
- Why WireGuard's design (silent to unauthenticated traffic) makes it a safer choice than traditional VPN protocols for a homelab exposed to any inbound port
- The value of testing every access-control change in a way that leaves a safety net (a second terminal session, an untouched fallback) before removing the old access path
---
title: "Building a 2-Node HA Kubernetes Cluster on Proxmox"
date: 2026-09-01
draft: false
tags: ["kubernetes", "k3s", "proxmox", "homelab"]
---

## Overview

I built a lightweight, highly-available Kubernetes cluster using [k3s](https://k3s.io/), running on two VMs inside a Proxmox homelab environment. The goal was to go beyond a single-node "toy" cluster and actually experience how Kubernetes handles multi-node control planes, workload scheduling, and failure recovery — the same concepts that matter in production infrastructure.

## Architecture

- **Hypervisor:** Proxmox VE, single physical host
- **VMs:** 2x Ubuntu Server 24.04 LTS (2 vCPU, 4GB RAM, 32GB disk each)
- **Kubernetes distro:** k3s with embedded etcd for control-plane HA
- **Networking:** Static IPs configured via netplan, both nodes on the same bridge
- **Redundancy model:** Node-level (VM failure), not hardware-level — since both VMs share one physical host, this simulates cluster behavior rather than true infrastructure redundancy

## Setting Up the Nodes

Each VM was provisioned with a static IP via netplan:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: no
      addresses: [192.168.1.101/24]
      routes:
        - to: default
          via: 192.168.1.1(this is your router IP address)
      nameservers:
        addresses: [8.8.8.8]
```

## Bootstrapping the Cluster

The first node was initialized as a k3s server with embedded etcd, which allows additional control-plane nodes to join later:

```bash
curl -sfL https://get.k3s.io | sh -s - server --cluster-init
```

The join token was retrieved from the first node:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

The second node joined as an additional server using that token:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://192.168.1.101:6443 \
  --token <TOKEN>
```

Both nodes were confirmed `Ready`:

```bash
sudo k3s kubectl get nodes
```

```
NAME           STATUS   ROLES                       AGE   VERSION
k3s-server1    Ready    control-plane,etcd,master    10m   v1.x.x
k3s-server2    Ready    control-plane,etcd,master    3m    v1.x.x
```

## Testing Redundancy

To confirm the cluster actually behaved as HA rather than just looking like it on paper, I deployed a test workload with 2 replicas:

```bash
sudo k3s kubectl create deployment mysite --image=nginx --replicas=2
sudo k3s kubectl expose deployment mysite --port=80 --type=NodePort
```

Kubernetes scheduled one pod per node automatically:

```bash
sudo k3s kubectl get pods -o wide
```

```
NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE
mysite-6d4b8f7c9d-abcde   1/1     Running   0          2m    10.42.0.5    k3s-server1
mysite-6d4b8f7c9d-fghij   1/1     Running   0          2m    10.42.1.6    k3s-server2
```

I then shut down one VM entirely and confirmed the site stayed reachable — the surviving node's pod kept serving traffic through the Service without any manual intervention.

## Challenges

**Netplan YAML indentation errors.** Static IP configuration failed initially due to inconsistent indentation (tabs mixed with spaces). Resolved by rewriting the file with consistent 2-space indentation and verifying with `cat -A` to catch hidden tab characters.

**Hostname not persisting after VM cloning.** Cloning a VM to quickly create the second node caused cloud-init to repeatedly reset the hostname back to the original on every boot, even after manual changes. Rather than fighting cloud-init's caching behavior, I opted for a clean OS install on the second node — a more reliable and reproducible approach for infrastructure that needs to be trustworthy.

## What I Learned

- How k3s's embedded etcd handles multi-node control-plane consensus
- The practical difference between node-level redundancy (what this project demonstrates) and hardware-level redundancy (what real production HA requires)
- Kubernetes' default pod scheduling behavior across nodes, and how Services route traffic regardless of which node a pod lands on
- The value of choosing a clean rebuild over prolonged debugging when a shortcut (cloning) introduces more complexity than it saves

## Next Steps

This site itself now runs on the cluster described above. Next up: containerizing it with Docker, deploying it as a Kubernetes Deployment/Service across both nodes, and exposing it through Traefik Ingress.
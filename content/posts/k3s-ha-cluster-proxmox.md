---
title: "Building a 2-Node HA Kubernetes Cluster on Proxmox"
date: 2026-09-01
draft: false
tags: ["kubernetes", "k3s", "proxmox", "homelab"]
---

## Overview

I built a lightweight, highly-available Kubernetes cluster using k3s, running on two VMs inside a Proxmox homelab environment. This project covers VM provisioning, networking, cluster bootstrapping, and workload redundancy testing.

## Architecture

- 2 Ubuntu Server 24.04 VMs (2 vCPU, 4GB RAM, 32GB disk each)
- k3s with embedded etcd for control-plane HA
- Static IP networking configured via netplan
- Workloads deployed with 2 replicas, scheduled across both nodes

## Key Steps

1. Provisioned VMs on Proxmox with static IPs
2. Bootstrapped the first k3s server node with `--cluster-init`
3. Joined a second server node using the cluster token
4. Verified both nodes reached `Ready` status
5. Deployed a test workload and confirmed pods scheduled across both nodes
6. Simulated a node failure by shutting down one VM — confirmed the site stayed reachable via the surviving node

## Challenges

- Netplan YAML indentation errors when configuring static IPs — resolved by ensuring space-only, consistent indentation
- Hostname not persisting correctly after cloning a VM (cloud-init overriding manual changes) — resolved by doing a clean install per node instead of cloning

## What I Learned

- How k3s embedded etcd handles multi-node control planes
- Practical differences between node-level and hardware-level redundancy
- Kubernetes' default pod scheduling behavior across nodes
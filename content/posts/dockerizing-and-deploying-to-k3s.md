---
title: "Dockerizing and Deploying This Site to My k3s Cluster"
date: 2026-09-03
draft: false
tags: ["docker", "kubernetes", "k3s", "hugo", "devops"]
---

## Overview

With the HA k3s cluster running, the next step was putting it to actual use: containerizing this Hugo portfolio site with Docker and deploying it as a Kubernetes workload spread across both cluster nodes.

## Building the Docker Image

Hugo compiles Markdown into static HTML, so the container just needs to build the site and serve the output — no runtime dependencies needed. I used a multi-stage Dockerfile: one stage builds the site with Hugo, the second serves it with nginx, keeping the final image small since it doesn't carry Hugo or Go tooling.

```dockerfile
# Stage 1: Build the Hugo site
FROM hugomods/hugo:exts AS builder
WORKDIR /src
COPY . .
RUN hugo --minify

# Stage 2: Serve with nginx
FROM nginx:alpine
COPY --from=builder /src/public /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Note:** I initially used the `klakegg/hugo` base image, which is no longer maintained and shipped a Hugo version incompatible with a newer theme I was using — the build failed with an unhelpful exit code. Switching to `hugomods/hugo`, the actively maintained successor, fixed it immediately. Good reminder to check whether a Docker base image is still maintained before troubleshooting your own code.

## Testing Locally

Before pushing anywhere, I built and ran the image locally to confirm it worked:

```bash
docker build -t my-portfolio:1.0 .
docker run -d -p 8080:80 --name portfolio-test my-portfolio:1.0
```

One gotcha here: Hugo bakes the `baseURL` value from `hugo.toml` into every internal link at build time. Testing on a different port than what `baseURL` specified caused internal navigation (About, Homelab Writeups) to point at the wrong address. Updating `baseURL` to match the actual serving address fixed it — a reminder that this value will need to change again once the site has its final Ingress hostname.

## Pushing to Docker Hub

```bash
docker login
docker tag my-portfolio:1.0 <dockerhub-username>/my-portfolio:1.0
docker push <dockerhub-username>/my-portfolio:1.0
```

## Deploying to the k3s Cluster

With the image public on Docker Hub, the cluster can pull it directly. I wrote a Deployment with 2 replicas and a Service to expose them internally:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portfolio
spec:
  replicas: 2
  selector:
    matchLabels:
      app: portfolio
  template:
    metadata:
      labels:
        app: portfolio
    spec:
      containers:
      - name: portfolio
        image: <dockerhub-username>/my-portfolio:1.0
        ports:
        - containerPort: 80
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: portfolio
spec:
  selector:
    app: portfolio
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

```bash
sudo k3s kubectl apply -f deployment.yaml
sudo k3s kubectl apply -f service.yaml
```

Kubernetes scheduled one pod per node automatically, same behavior confirmed in the original cluster writeup:

NAME                        READY   STATUS    NODE
portfolio-5d9f6f9bf-45g8f   1/1     Running   k3s-server2
portfolio-5d9f6f9bf-rhghg   1/1     Running   k3s-server1


I used `ClusterIP` rather than `NodePort` here since the plan is to expose the site properly through Traefik Ingress next, rather than relying on a raw node port.

## What I Learned

- How Docker multi-stage builds keep final images minimal by discarding build-time tooling
- Why base image maintenance status matters — an abandoned image caused a confusing build failure that had nothing to do with my actual code
- How `baseURL` in a static site generator ties into deployment environment, and why it needs to be revisited at each stage (local test → cluster → final domain)
- The full path from source code to a running, redundant Kubernetes workload: build → registry → manifest → cluster

## Next Steps

The site is now running on the cluster, but only reachable internally via ClusterIP. Next: setting up Traefik Ingress so it's reachable by hostname from a browser, the same way a real deployed service would be.
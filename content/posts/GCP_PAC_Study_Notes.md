---
title: "GCP Professional Architect Certification — Study Notes"
author: "M. Azam"
date: 2026-05-19
draft: false
tags: ["gcp", "certification", "PAC", "study-notes"]
categories: ["Tutorial"]
description: "Random notes on GCP PAC certifcation preperation"
showToc: true
---

# GCP Load Balancers

A Load Balancer in GCP is a managed service that distributes incoming traffic across multiple backend services to ensure availability and scalability. It is composed of several components that work together:

1. **Static IP Address** — The entry point for all traffic (external or internal VPC).
2. **Forwarding Rule** — Binds the static IP, port, and protocol to the proxy. It directs incoming traffic to the correct proxy.
3. **Proxy (Reverse Proxy) — The core of the LB. It receives traffic from the forwarding rule and coordinates SSL/TLS and URL mapping.
4. **SSL/TLS** — Decrypts incoming HTTPS traffic so it can be processed and forwarded to the backend.
5. **URL Mapper — A configuration that routes traffic to the correct backend based on the hostname and URL path.
6. **Backend Services** — The actual destinations for traffic, which can be VMs (in Managed Instance Groups), containerized apps (Cloud Run or GKE), or Backend Buckets (pointing to Cloud Storage).
7. **Health Checks** — The LB periodically pings each backend instance to determine if it is healthy. Unhealthy instances are removed from the traffic pool.
8. **Load Balancing Algorithm** — Determines which specific healthy backend instance receives each request (e.g. round-robin).

## GCP Load Balancer Types

GCP offers Load Balancers at two layers of the network stack:

### Application Load Balancer (Layer 7)

- Handles HTTP and HTTPS traffic
- Can inspect request content — headers, URL path, hostname
- Makes intelligent routing decisions based on domain and URL path (via URL mapper)
- Supports integration with Cloud Armor (WAF, security rules)
- Available as External (public internet traffic with public static IP) or Internal (within VPC using private IP)
- Primarily Global — can route traffic across regions for failover and performance

### Network Load Balancer (Layer 4)

- Handles TCP and UDP traffic
- Can only see IP address and port — no visibility into content
- Faster and simpler routing but with less control
- Available as Regional or Global
- Used when the protocol is not HTTP/HTTPS or when low-latency raw traffic handling is needed

**Key Decision Rule:**

- Use Application LB when you need content-based routing, security filtering, or HTTP/HTTPS with rich control
- Use Network LB when you are dealing with TCP/UDP protocols that require fast, simple routing

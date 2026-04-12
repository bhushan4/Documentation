# Ingress vs Egress: Network Traffic Direction

A quick reference for understanding traffic direction terminology used across Kubernetes, cloud infrastructure, firewalls, and networking in general.

---

## The Two Directions

**Ingress** means traffic coming *into* your system from the outside world. The word literally means "the act of entering." A user's browser hitting your API, a webhook arriving from GitHub, a client uploading a file — all ingress.

**Egress** means traffic going *out* from your system to the outside world. The word literally means "the act of exiting." Your pod calling Stripe's API, sending an email via SendGrid, fetching data from a third-party service — all egress.

```
User's browser  ──────→  Your cluster     =  Ingress (incoming)
Your cluster    ──────→  Stripe API        =  Egress (outgoing)
Pod A           ──────→  Pod B (internal)  =  Neither — east-west traffic
```

Internal traffic between pods within the same cluster is typically called **east-west traffic** — it's neither ingress nor egress because it never leaves the cluster boundary. The ingress/egress distinction only applies when traffic crosses the boundary between your system and the outside world.

---

## Where These Terms Show Up

### Network Security Groups (AWS, Azure, GCP)

Every security group or firewall rule is split into two sections:

- **Ingress rules** control who can send traffic *to* your resource. Example: "Allow TCP port 443 from any IP" lets HTTPS traffic in.
- **Egress rules** control where your resource can send traffic *to*. Example: "Allow TCP port 5432 to the database subnet" lets your app reach the database.

By default, most cloud security groups allow all egress but deny all ingress. You explicitly open ingress rules for the ports you need.

### Firewalls

Same concept. A corporate firewall has ingress rules (what external traffic is allowed in) and egress rules (what internal traffic is allowed out). Egress filtering is how companies block access to specific websites or prevent data exfiltration.

### Cloud Billing

This is a real cost consideration at scale:

- **Ingress is usually free.** Cloud providers don't charge you for receiving data. They want data to flow into their network.
- **Egress costs money.** Sending data out of the cloud provider's network is charged per GB. AWS, GCP, and Azure all charge for egress. At high volumes (terabytes/month), egress can be one of the largest line items on your cloud bill.
- **Cross-region and cross-AZ traffic** also incurs egress-like charges even within the same cloud provider, because the data crosses a network boundary.

### Kubernetes Specifically

In Kubernetes, these terms appear in two contexts:

1. **General networking** — ingress/egress as described above, referring to any traffic entering or leaving the cluster.

2. **The Ingress API object** (capital I) — a specific Kubernetes resource that manages HTTP ingress traffic. It routes external requests to the right internal Service based on hostname and URL path. For example: requests to `api.mycompany.com/users` go to the users Service, requests to `api.mycompany.com/orders` go to the orders Service. This is what sits at the front door of your cluster for HTTP traffic.

3. **NetworkPolicy** — a Kubernetes resource that controls pod-level ingress and egress. You can write rules like "Pod A can only receive ingress from pods with label `app: frontend`" or "Pod B can only send egress to port 5432 on pods with label `app: postgres`." This is how you implement micro-segmentation and zero-trust networking inside the cluster.

---

## Quick Reference

| Term | Direction | Example | Cost |
|---|---|---|---|
| Ingress | Outside → your system | User hits your API | Usually free |
| Egress | Your system → outside | Your app calls Stripe | Charged per GB |
| East-west | Internal pod → pod | Frontend calls backend | Free (same AZ) or cheap |

---

*Last updated: April 2026*
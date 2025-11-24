# Azure AKS Architecture - Interview Guide

**Time Allocation**: 15 Minutes
**Goal**: Demonstrate expertise in Cloud-Native design, Security, and Cost Optimization.

---

## 1. The "Elevator Pitch" (1 Minute)
*Start with a high-level summary to set the stage.*

"I designed a **Cloud-Native, Secure, and Cost-Efficient** platform on Azure to host our three core services: `api-gateway`, `auth-service`, and `reports-service`.
The architecture leverages **Azure Kubernetes Service (AKS)** as the compute engine, wrapped in a **Zero-Trust network** using VNETs and Private Links.
I prioritized **resilience** by spreading workloads across zones and **cost efficiency** by using Spot instances for non-critical workloads."

---

## 2. The "Front Door": Traffic Flow & Security (3 Minutes)
*Point to the top of the diagram (User -> DNS -> AppGW).*

**Key Components:**
- **Azure DNS / Traffic Manager**: The entry point.
- **Application Gateway (WAF) + AGIC**:
    - **Why?** It acts as our Ingress Controller. It provides Layer 7 load balancing and, crucially, a **Web Application Firewall (WAF)** to block attacks (SQL injection, XSS) at the edge before they reach our cluster.
    - **AGIC (App Gateway Ingress Controller)**: This allows us to configure the gateway directly from Kubernetes Ingress resources, bridging the gap between Devs and Ops.

**Talking Point**: "I chose App Gateway over a standard Load Balancer because we need SSL termination and WAF protection for our public-facing SaaS."

---

## 3. The "Heart": AKS Cluster & Node Strategy (5 Minutes)
*Point to the `AKS Cluster` box. This is where you show off your optimization skills.*

**The Cluster Design**:
"I segmented the cluster into **three distinct Node Pools** to isolate workloads and optimize costs:"

1.  **System Pool (Critical)**:
    - **What runs here?** CoreDNS, Metrics Server, Ingress Controllers.
    - **Why?** These are the cluster's brain. We isolate them so that a memory leak in a user app doesn't crash the entire cluster.
    - **Configuration**: Stable, reserved instances. Tainted so only system pods run here.

2.  **User Pool (General Workloads)**:
    - **What runs here?** `api-gateway`, `auth-service`.
    - **Why?** These are 24/7 critical services.
    - **Configuration**: Spread across Availability Zones (AZs) for High Availability. If one zone goes down, our auth service stays up.

3.  **Spot Pool (Batch/Background)**:
    - **What runs here?** `reports-service`.
    - **Why?** Reporting is often bursty and can tolerate interruptions.
    - **The "Pro" Move**: I used **Azure Spot Instances** here, which are up to **90% cheaper**. I configured `tolerations` so only the report service runs here, and implemented graceful shutdown logic to handle Spot evictions.

**💡 Important Clarification**:
"In the diagram, you'll see the `api-gateway` and `auth-service` in their own box. This is the **User Node Pool** (or Application Pool). I separated this from the System Pool to ensure that heavy user traffic never starves the critical cluster components (like CoreDNS)."

---

## 4. The "Vault": Data & Private Networking (3 Minutes)
*Point to the `Data Subnet` and `VNET` boundaries.*

**Security First**:
- **VNET Integration**: The entire cluster sits inside a private VNET. Nothing is exposed to the public internet except the App Gateway.
- **Private Link**:
    - **Azure SQL & Redis**: These are NOT accessible over the public internet. We use **Private Endpoints** to inject them directly into our VNET.
    - **Benefit**: Traffic between our apps and the database never leaves the Azure backbone network. It's faster and much more secure.

**Identity**:
- **Workload Identity**: Instead of managing secrets and passwords for the database, I used **Azure Workload Identity**. The pods authenticate to SQL and Key Vault using their Kubernetes Service Account (OIDC). No hardcoded passwords!

---

## 5. Observability & Operations (3 Minutes)
*Point to `Azure Monitor` and `ACR`.*

- **Container Registry (ACR)**: Stores our Docker images. We scan images for vulnerabilities here before deployment.
- **Azure Monitor / Container Insights**:
    - We scrape metrics (CPU, Memory) and logs (stdout/stderr).
    - I set up **Prometheus** metrics for application-level insights (e.g., "Login Latency", "Report Generation Time").
    - **Alerts**: We alert on "Golden Signals": Latency, Traffic, Errors, and Saturation.

---

## Summary / Closing (1 Minute)
"In summary, this architecture balances **Performance** (via dedicated node pools), **Security** (via WAF and Private Link), and **Cost** (via Spot instances). It allows the business to scale the `reports-service` cheaply while keeping the critical `auth-service` highly available."

---

## 6. Addressing "Missing" Parts (Staff Level Discussion)
*Since you are submitting the Diagram + Terraform, use this section to discuss what you WOULD build next. This shows leadership and vision.*

**Q: How will you deploy applications?**
"I would implement a **GitOps** workflow using **ArgoCD**.
- It provides a single source of truth (Git).
- It automatically detects drift.
- I'd use a 'App of Apps' pattern to manage the Helm charts for our 3 services."

**Q: How do we monitor this?**
"I'd use **Azure Managed Prometheus** and **Grafana**.
- **SLOs**: I would define SLOs for the `api-gateway` (e.g., 99.9% availability, <200ms latency).
- **Tracing**: I'd insist on **OpenTelemetry** instrumentation in the code to trace requests from the Gateway -> Auth -> DB."

**Q: How do we secure service-to-service communication? (Crucial for Staff Level)**
"Currently, the diagram shows network segmentation. The next step is a **Service Mesh** (like Istio or Linkerd).
- **mTLS**: This would encrypt all traffic between pods (e.g., Gateway -> Auth).
- **Identity**: It ensures only the Gateway is *allowed* to talk to Auth, enforcing a strict Zero Trust policy."

**Q: How do we handle the `reports-service` being on Spot instances?**
"The application needs to be 'Spot Aware'.
- It should listen for the `SIGTERM` signal (Kubernetes gives a 30s warning before eviction).
- Upon receiving it, the app should checkpoint its work and exit gracefully so a new pod can resume the job on another node."

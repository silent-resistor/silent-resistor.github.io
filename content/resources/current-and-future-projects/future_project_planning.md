---
title: "Real-World Phishing & Edge Infrastructure Platform"
date: 2026-10-12
weight: 1
---

## Project Roadmap
### Phase 1: Cloud Provisioning & Baseline Server Setup (Days 1 to 5)
- **Goal**: Acquire a permanent zero-cost cloud instance and configure core system security and networking.
- **Cloud Account & Instance Provisioning (Days 1–2)**: Register for an Oracle Cloud Infrastructure (OCI) Always Free account. Provision an Always Free ARM compute instance (VM.Standard.A1.Flex with 2 OCPUs and 12 GB RAM) running Ubuntu Linux and assign a static public IP.
- **Network & Firewall Hardening (Days 3–4)**: Configure OCI cloud security lists and local firewall rules (ufw) to restrict traffic strictly to necessary ports: 22 (SSH), 80 (HTTP), and 443 (HTTPS).
- **Base Stack Installation (Day 5)**: Install core container runtimes and web management tools (Docker, Docker Compose, Git, and Nginx) on the instance.

### Phase 2: Edge Redirector & Evasion Middleware (Days 6 to 12)
- **Goal**: Build the custom traffic controller/bouncer that sits at the edge to filter out security scanners, automated crawlers, and bots.
- **Reverse Proxy & TLS Configuration (Days 6–7)**: Install Certbot, generate a valid Let's Encrypt SSL certificate, and configure Nginx to terminate TLS on port 443. Set Nginx up to forward requests internally to your middleware application.
- **Middleware Development (Days 8–9)**: Initialize a FastAPI or Go project to capture and log every incoming request's metadata, including client IP address, User-Agent strings, and core HTTP headers (Accept-Language, Sec-Fetch-Dest).
- **Threat & Bot Filtering Logic (Days 10–11)**: Implement logic to cross-reference visitor IPs against known cloud provider ranges and detect headless browsers or command-line tools (curl, python-requests).
- **Dynamic Routing Integration (Day 12)**: Write conditional branching rules: scanners/bots get served a standard HTTP 404 or a static clean template, while verified targets are silently proxied forward.

### Phase 3: Campaign Management & Backend Staging (Days 13 to 19)
- **Goal**: Deploy the campaign manager safely behind your edge bouncer and configure payload delivery workflows.
- **Isolated GoPhish Deployment (Days 13–14)**: Deploy GoPhish using Docker Compose. Bind it strictly to an internal port or loopback interface so it is completely shielded from direct public exposure.
- **Landing Page & Staging Configuration (Days 15–17)**: Configure GoPhish landing pages to host mock corporate portals or download staging assets (such as a dummy test dropper or document simulator).
- **End-to-End Pipeline Integration (Days 18–19)**: Route verified traffic from your Nginx edge redirector straight into the internal GoPhish container instance, ensuring that direct hits to the backend are blocked.

### Phase 4: Operational Verification & Documentation (Days 20 to 25)
- **Goal**: Validate operational security (OpSec) controls and package the project structure for review.
- **Traffic Simulation Testing (Days 20–22)**: Execute tests using automated scripts versus real browsers to confirm that your edge bouncer successfully catches and drops bots while letting clean sessions pass.
- **Infrastructure Cleanup & Automation (Days 23–24)**: Write cleanup scripts or simple automation templates to ensure disposable nodes can be decommissioned cleanly without leaving tracking artifacts.
- **Documentation & Architecture Mapping (Day 25)**: Create a clear traffic-flow diagram and write a comprehensive project README.md detailing your edge filtering rules and architectural defense layers.

    ![Road map](/resources_projects/phishin_infrastructure_map.jpeg)

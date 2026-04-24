# Reverse Proxy vs. Load Balancer vs. API Gateway

Three components that sit between clients and servers — they all "forward requests" but solve fundamentally different problems.

**The core problem:** As systems grow, you need ways to handle more traffic, secure your backend, and manage multiple services.

---

## 1. Forward Proxy vs. Reverse Proxy

### Forward Proxy (Client-Side)
- Sits in front of **clients**
- Acts as a middleman between a private network and the public internet
- **Key functions:**
  - **Anonymity** — hides the client's IP from the destination server
  - **Filtering** — blocks harmful sites or restricted content (common in corporate environments)
  - **Caching** — stores common responses locally to save bandwidth

### Reverse Proxy (Server-Side)
- Sits in front of **servers**
- Protects and manages backend servers; clients interact with the proxy, not the server directly
- **Key functions:**
  - **Server anonymity** — hides backend server details (IPs, architecture) from the public
  - **SSL termination** — handles decrypting/encrypting HTTPS traffic
  - **Compression** — compresses outbound responses (e.g., GZIP) to speed up delivery
  - **Caching** — serves static content (images, videos) without hitting the backend
- **Common tools:** Nginx, HAProxy, Caddy, Apache

---

## 2. Load Balancer

A specialized reverse proxy with one specific mission: **distribution**.

- **Triggered when** one server can no longer handle the traffic volume
- **Primary goals:**
  - **Scalability** — handle more traffic by adding more servers
  - **Availability** — if one server fails, route traffic to healthy servers (health checks)
- Ensures no single server becomes a bottleneck or single point of failure
- **Common tools:** AWS ALB/NLB, GCP Load Balancer, F5

---

## 3. API Gateway

An API-aware proxy designed for microservices and web APIs. Handles **cross-cutting concerns** so developers don't have to code them into every service.

- **Key functions:**
  - **Authentication/Authorization** — centralizes login and permissions at the edge
  - **Rate limiting** — prevents abuse by capping requests per client
  - **Request transformation** — converts protocols (e.g., JSON to XML) or modifies headers
  - **Versioning** — routes traffic to different API versions (`/v1/` vs `/v2/`)
  - **Monitoring** — tracks latency (P95) and usage analytics
- **Common tools:** Kong, AWS API Gateway, Apigee, Tyk

---

## 4. Why the Confusion? (The Overlap)

Modern tools often perform multiple roles:

- **Nginx** is a reverse proxy, but can also do load balancing
- **Kong** is an API gateway, but is built on top of Nginx
- **Cloud providers** (AWS, etc.) sell these as separate products, but capabilities overlap (e.g., an ALB can do basic content-based routing)

---

## 5. Comparison Table

| Feature | Reverse Proxy | Load Balancer | API Gateway |
|---|---|---|---|
| **Primary focus** | Security & Performance | Scale & Availability | API Management |
| **Acts on behalf of** | The Server | The Server Fleet | The API / Microservice |
| **Typical tasks** | SSL, Caching, Compression | Traffic distribution, Health checks | Auth, Rate limiting, Versioning |
| **Nature** | General Purpose | Specialized | API-Specific |

---

## 6. Real-World Architecture

In production, these are often **layered**:

1. **Load Balancer** — receives initial traffic and distributes it to a cluster of gateways
2. **API Gateway** — handles authentication and determines which microservice gets the request
3. **Reverse Proxy** — may sit in front of an individual service for specific caching or SSL needs

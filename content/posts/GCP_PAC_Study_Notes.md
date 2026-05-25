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

# GCP Concepts for PAC prepration

## GCP Load Balancers

A Load Balancer in GCP is a managed service that distributes incoming traffic across multiple backend services to ensure availability and scalability. It is composed of several components that work together:

1. **Static IP Address** — The entry point for all traffic (external or internal VPC).
2. **Forwarding Rule** — Binds the static IP, port, and protocol to the proxy. It directs incoming traffic to the correct proxy.
3. **Proxy (Reverse Proxy)** — The core of the LB. It receives traffic from the forwarding rule and coordinates SSL/TLS and URL mapping.
4. **SSL/TLS** — Decrypts incoming HTTPS traffic so it can be processed and forwarded to the backend.
5. **URL Mapper** — A configuration that routes traffic to the correct backend based on the hostname and URL path.
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

## GCP Data Storage Summary

### Structured Data (Relational)

- **Cloud SQL** — Managed MySQL/PostgreSQL, regional, suitable for standard business transactional workloads

- **Cloud Spanner** — Global, multi-region, low latency relational database, but significantly more expensive. Suited for large global businesses

### Semi-Structured / Document

- **Firestore** — Document-based (JSON-like) storage with nesting support. Ideal for web/mobile apps needing real-time updates (e.g. order status). Not suited for high-frequency or large-scale data

### Wide-Column / High Volume

- **BigTable** — Suited for enormous volumes of data with high write throughput. Ideal for IoT, sensor data, and time-series data where transactional integrity is not a priority

### In-Memory Cache

- **Memorystore (Redis)** — Caches frequently accessed, rarely changing data to reduce database load. Uses TTL and event-driven invalidation to manage stale data

### Object Storage

- **Cloud Storage** — Stores BLOBs (images, video, files) across four classes: Standard, Nearline, Coldline, and Archive — balancing storage cost vs retrieval frequency

### Analytical / Data Warehouse

- **BigQuery** — Suited for large-scale analytical queries (OLAP), not real-time transactions. Ideal for trend analysis across large historical datasets


## Kubernetes & GKE Summary

### Containers

- Lightweight isolated processes sharing the host OS
- Unlike VMs, containers don't need a full OS — making them fast and lightweight
- Eliminate dependency conflicts between test and production environments

### Pods

- A wrapper around one or more containers
- Tightly coupled containers share the same pod
- Each pod has a dynamic, ephemeral IP address

### Services

- Groups pods together under a stable static IP address
- Pods register their IP with the Service when they start
- Acts as a stable endpoint so the Load Balancer can route traffic reliably

### Kubernetes Architecture

![alt text](/post/image.png)

- Control Plane manages the cluster:
  - API Server — gateway between users/tools (gcloud, console) and the cluster
  - Scheduler — places pods on nodes based on available CPU/memory resources
  - Controller Manager — the "cop" that ensures the desired state is maintained (e.g. restarting crashed pods)
  - etcd — stores the entire state of the cluster

- Worker Nodes run the workloads:
  - kubelet — agent communicating with the control plane
  - kube-proxy — manages networking and routing between pods across nodes
  - Container runtime — actually runs the containers (e.g. containerd)

NOTE: Here's a quick reference link from the official GCP docs for the architecture: GKE Cluster Architecture

### GKE (Google Kubernetes Engine)

- Google's managed Kubernetes service
- Google manages the control plane in both modes
- Standard mode — user manages nodes (manual scaling via kubectl)
- Autopilot mode — Google manages nodes, inferring VM configuration from pod resource requirements

**Summary of the Kubernetes vs GKE distinction**:

**Kubernetes** is the open-source container orchestration platform. It can run anywhere — on your own servers, on AWS, Azure, or GCP. When you run vanilla Kubernetes, you are responsible for everything: setting up the control plane, managing nodes, networking, upgrades, and security.

**GKE** is Google's managed Kubernetes service. It uses Kubernetes under the hood but removes much of the operational burden. Google manages the control plane — running the API server, scheduler, controller manager, and etcd — in both Standard and Autopilot modes. [Google Cloud](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture)

The two modes differ in how much Google takes over beyond the control plane:

In **Standard mode**, Google manages the control plane and system components, but users manage the nodes. [Medium](https://medium.com/@chatterjee.mithun/autopilot-is-now-gkes-default-mode-of-operation-here-s-what-that-means-for-you-514a02357713)

In **Autopilot mode**, GKE provisions and manages the corresponding infrastructure to run your workloads based on what you specify in your workload definitions. [Google](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)

The key billing distinction is that in Standard mode you pay for nodes whether you use them or not, while in Autopilot you pay only for the CPU, memory, and storage your pods actually request.

## GCP IAM Summary

### Organization Hierarchy

- **Organization** — root node, establishes identity boundary for the org
- **Folders** — logical grouping of projects (by department, environment, etc.)
- **Projects** — the only entity that can own GCP resources; linked to a billing account
- **Resources** — compute, storage, etc.; live inside projects

### Principals (Identities)

- **Users** — human identities, either within org or external
- **Groups** — collection of users sharing the same permissions
- **Service accounts** — non-human identities used by services and applications

### Roles

- **Basic/Primitive** — broad roles: owner, editor, viewer
- **Predefined** — fine-grained roles created and maintained by Google per service
- **Custom** — organization-defined roles combining specific permissions

### Policies

- A policy binds a principal + role + resource
- Policies are additive — principals accumulate permissions across all levels
- Policies are inherited down the hierarchy — org → folder → project → resource
- Deny policies always take precedence over allow policies

### Service Accounts

- Can act as a principal — when a service uses it to access resources
- Can act as a resource — when a human is granted permission to impersonate it
- Default service accounts are broad — Google recommends custom service accounts following least privilege
- Service account keys — downloadable credentials, discouraged due to security risks

### Workload Identity Federation

- Allows external services (like GitHub) to authenticate to GCP without service account keys
- External identity provider (e.g. GitHub) issues a signed JWT token
- GCP validates the token and allows impersonation of a designated service account
- Much more secure than storing service account keys in external systems

### Super User / Organization Admin

- In GCP the "super user" concept is represented by the Organization Admin role
- The org admin can grant and manage IAM roles but has no direct access to resources — enforcing least privilege even at the highest level
- The first org admin is bootstrapped from the Google Workspace or Cloud Identity account used to create the organization
- Risks: a compromised org admin can indirectly cause damage by granting elevated privileges to malicious principals
- Safeguards: multiple org admins, audit logging, and narrowly scoped admin roles

## GCP Data Security

![alt text](/post/image-1.png)

### Encryption at Rest

- **Google managed (default)** — GCP automatically encrypts data with a DEK
- **CMEK (Customer Managed Encryption Key)** — user creates KEK in Cloud KMS; GCP wraps DEK with KEK
- **CSEK (Customer Supplied Encryption Key)** — user passes key bytes in request header; Google never stores or manages the key
- **Cloud HSM** — keys stored in tamper-resistant hardware devices; even Google cannot access them; required for strict government and industry compliance (FIPS 140-2)

### Key Management

- **DEK (Data Encryption Key)** — encrypts actual data
- **KEK (Key Encryption Key)** — wraps and protects the DEK
- **Cloud KMS** — Google's managed key management service for KEKs
- **Cloud HSM** — hardware backed key storage for maximum security and compliance

### Encryption in Transit

- User to GCP (internet) — protected by TLS:
  - Asymmetric encryption (public/private key) used during handshake to verify identity via certificates and negotiate a shared secret
  - Symmetric encryption (shared session key) used for actual data exchange — faster and efficient
  - Certificates issued by trusted Certificate Authorities (CAs)

- Within GCP internal network — Google encrypts all data in transit between data centers and services automatically by default

### Secret Manager

- Stores secrets (passwords, tokens, API keys, certificates) in a centrally managed, secure service
- Protected by IAM policies — only authorized principals can access secrets
- Supports versioning — enables secret rotation, supports multiple API versions, and maintains audit trails
- Recommended pattern: application uses its service account to retrieve secrets from Secret Manager at runtime — avoids storing credentials in environment variables or config files

### Sensitive Data Protection (Cloud DLP)

- Automatically inspects data to detect sensitive fields (credit card numbers, SSNs, EHR data)
- **De-identification techniques**:
  - **Masking** — replaces sensitive value with ****
  - **Tokenization** — replaces sensitive value with a consistent token, preserving correlation across records
  - **Generalization** — replaces exact value with a range (e.g. age 67 → 60-70)
  - **Bucketing** — groups values into ranges (e.g. salary 60k-75k)
  - **Redaction** — completely removes the sensitive field
- **Addresses re-identification risk** — combination of seemingly harmless fields can still identify a person

### VPC Service Controls

- Creates a security perimeter around GCP resources
- Prevents sensitive data from leaving the perimeter even if attacker has valid credentials
- Ideal for highly regulated data like EHR — ensures data cannot leave the defined boundary

### Access Control

#### Context-Aware Access

- Goes beyond identity to verify contextual attributes before granting access:
  - **Device identity** — is it a registered corporate device?
  - **OS version** — is it up to date and patched?
  - **Location** — is the user within an approved location?
  - **Network** — is the request coming from an approved IP address?
  - **Device security** — screen lock, storage encryption
- **Implements zero trust security** — never trust, always verify

#### Identity-Aware Proxy (IAP)

- A fully managed GCP service that sits as a proxy between the user and the application
- Verifies identity and IAM policies on every request before forwarding to the application
- Advantages over VPN:
  - Access granted per application, not entire network — least privilege
  - Identity verified on every request, not just at connection time
  - No VPN client needed — works over HTTPS
  - Integrates directly with GCP IAM

### Context-Aware Access + IAP — Zero Trust Model

  - IAP answers: who are you? — identity and IAM verification
  - Context-Aware Access answers: are your circumstances acceptable? — device, location, network verification
  - Together they ensure that even a valid user with valid credentials is denied access if contextual conditions are not met

## Google Cloud AI Products

### The 4-Layer AI Framework

Google organizes its AI offerings into 4 layers, ordered from **least to most effort**, where more effort also gives **more control**.

#### Layer 1 — Pre-built APIs
**Effort**: Lowest | **Control**: Lowest

- Ready-to-use, pre-trained models accessible via REST API
- No training data, no ML expertise required
- Google has already trained the model — you just call the API

**Services include:**
- **Vision AI** — image labeling, object detection, OCR
- **Video Intelligence API** — recognizes objects, scenes, and actions in existing video (analysis only, not generation)
- **Speech-to-Text** — audio transcription
- **Text-to-Speech** — convert text to spoken audio
- **Translation API** — language translation
- **Document AI** — structured data extraction from forms and invoices
- **Natural Language API** — sentiment analysis, entity detection

**Limitation:** Only works for use cases Google has already trained models for (common, well-known domains).

---

#### Layer 2 — BigQuery ML
**Effort**: Low-Medium | **Control**: Medium-Low

- Train ML models using **SQL** directly where your data lives in BigQuery
- No data movement, no Python or ML coding required
- Target user: **data analysts** familiar with SQL
- Uses extended SQL commands like `CREATE MODEL` and `ML.PREDICT`

**Best for:** Users who already have structured data in BigQuery and want to build ML models without learning Python or ML frameworks.

---

#### Layer 3 — AutoML (Vertex AI)
**Effort**: Medium | **Control**: Medium

- User brings **labeled training data**
- Google handles **model architecture, hyperparameter tuning, and training**
- No deep ML expertise required — Google abstracts the hard ML engineering

**Services include:**
- AutoML Vision
- AutoML Video
- AutoML Text
- AutoML Tables

**Key concept — Hyperparameters:** Configuration settings that control *how* a model is trained (e.g. learning rate, number of iterations, model complexity). Google manages these automatically in AutoML.

**Best for:** Use cases where pre-built APIs don't work — e.g. new undiscovered species, aboriginal languages, proprietary products not on the internet.

---

#### Layer 4 — Vertex AI Custom Training
**Effort**: Highest | **Control**: Highest

- User writes **their own training code** using frameworks like TensorFlow, PyTorch, scikit-learn, etc.
- User is responsible for **model architecture, hyperparameter tuning, and fine-tuning**
- Google provides only **managed infrastructure and tools**
- Requires deep ML and domain expertise

**Best for:** Data scientists and ML engineers who need full control over the training process for highly specialized requirements.

---

### Vertex AI Model Garden & Foundation Models

A newer Google offering focused on **Generative AI** — adds a new dimension beyond the 4 layers.

#### What it provides:
- Access to **Google's foundation models:**
  - **Gemini** — general purpose LLM (chat, reasoning, multimodal)
  - **Imagen** — text-to-image generation
  - **Chirp** — audio-to-text transcription
- Access to **open source models** hosted and managed by Google:
  - Llama, Mistral, and others

#### Key concepts:

**Foundation Models vs Pre-built APIs:**
- Pre-built APIs are task-specific services (vision, translation, etc.)
- Foundation models are large general-purpose models for generative AI use cases

**Fine-tuning vs AutoML:**
| | Fine-tuning (Model Garden) | AutoML (Layer 3) |
|---|---|---|
| Starting point | Pre-trained foundation model (e.g. Gemini) | No pre-existing model for the use case |
| Data needed | Less labeled data needed | More labeled data required |
| Time to market | Faster | Slower |
| Best for | Extending what Gemini already knows | Completely new use cases |

**Why open source models in Model Garden?**
- Without Model Garden, users would need to download, host, and manage models themselves (on a laptop or VM)
- Google provides: managed infrastructure, scalability, security, and a single unified platform
- Users get flexibility to choose Google or open source models in one place

#### Example use cases:
- **Gemini** → general purpose chatbot for a website
- **Chirp** → audio transcription for restaurant orders
- **Imagen** → generating product images or book covers
- **Llama/Mistral** → open source alternatives hosted on Google infrastructure

---

### Quick Reference: Which Layer to Use?

| Scenario | Layer |
|---|---|
| Analyze images, detect objects, translate text | Layer 1 — Pre-built APIs |
| Analyze video content (scenes, objects) | Layer 1 — Video Intelligence API |
| Have data in BigQuery, comfortable with SQL | Layer 2 — BigQuery ML |
| Have unique labeled data, minimal coding preferred | Layer 3 — AutoML |
| Need full control, write own training code | Layer 4 — Vertex AI Custom Training |
| Generative AI, extend existing foundation models | Vertex AI Model Garden |

### Securing AI/ML Model

#### AI/ML Workload Components

Before securing anything, we established the four key components of an AI/ML workload:

1. Training data — labeled data to teach the model plus validation data to verify accuracy
2. Model — the algorithm (blueprint) plus learned weights (probability distributions)
3. Configuration — hyperparameters (how training works) plus guardrails (constraints and boundaries)
4. Prediction traffic — real-time inference data (input) and predictions (output)

We also clarified that the algorithm and the trained model are different things — the algorithm is the recipe, and the trained model is the finished product after training.

---

#### Securing Training Data

- GCP encrypts data at rest and in transit by default
- CMEK (Customer Managed Encryption Keys) via Cloud KMS for user managed keys
- CSEK (Customer Supplied Encryption Keys) for users who supply their own key bits per request
- Cloud HSM for tamper resistant hardware based key management
- Sensitive Data Protection for anonymizing sensitive data through redaction, masking, bucketing, tokenization
- Artifact Registry for verifying integrity of libraries and container images through code signing
- Data poisoning defense through IAM access restrictions, Cloud Storage object versioning, hash verification, and Cloud Audit Logs

---

#### Securing the Model

- IAM with least privilege principle controls who can access the model
- Service accounts represent ML workloads (not humans) when accessing GCP resources
- Identity-Aware Proxy (IAP) protects HTTP based model endpoints
- VPC Service Controls creates a security perimeter preventing data exfiltration — complementing IAM by controlling where data can go, not just who can access it
- Context-Aware Access adds an additional layer considering device state, OS version, location, IP address, and network trustworthiness

---

#### Securing Configuration

- All access control strategies above apply equally to configuration data
- Cloud Audit Logs provides immutable audit trails tracking who changed what and when
- Three types of audit logs: Admin Activity (always on, cannot be disabled), Data Access, and System Event logs
- Audit logs can be configured at organization, folder, or project level — organization level recommended for full visibility
- Accessible via IAM & Admin in the GCP console

---

#### Securing the Training Pipeline

- Secret Manager secures API keys and database credentials used by ML workloads, avoiding hardcoded credentials in code
- Workload Identity Federation secures CI/CD deployment pipelines by eliminating long-lived service account key files, replacing them with short-lived tokens exchanged via JWT — preventing accidental credential exposure in repositories like GitHub

---

#### Securing Prediction Traffic

- Cloud Armor — protects model endpoints at the network level against DDoS attacks, bot traffic, malicious IPs, and common web application attacks (SQL injection, cross-site scripting). It is the first line of defense before any request reaches your model.


---

#### Securing the Model — Additional Layer

- Model Armor — a fully managed GCP service that acts as an AI-specific firewall, screening both incoming prompts and outgoing model responses. It protects against:
  - Prompt injection — attackers crafting malicious inputs to manipulate model behavior
  - Jailbreaking — attempts to bypass the model's safety guidelines
  - Sensitive data leakage — preventing the model from revealing PII or proprietary information in its responses
  - Harmful content — filtering hate speech, harassment, and dangerous topics
  - Malicious URLs — detecting harmful links embedded in prompts or responses



Together, Cloud Armor and Model Armor implement a defense in depth strategy — Cloud Armor at the network level and Model Armor at the AI application level, each catching what the other might miss.

### GDPR & Data Residency on GCP

GDPR Scope — The Key Principle: GDPR Follows the Subject

#### GDPR applies based on the location of the data subject, not citizenship
- EU residents (any nationality) → GDPR applies
- Non-EU citizen physically in EU when data collected → GDPR applies
- Any company worldwide processing EU residents' data → GDPR applies
- EU citizen living outside EU → GDPR does NOT automatically apply

KnightMotives — a fictional company scenario used in study material to provide real world GDPR context.

#### Key GDPR Principles relevant to GCP:
- Data Sovereignty — data is subject to laws of the region where it is stored
- Right to Erasure — users can request their data to be deleted
- Data Minimization — collect only what is necessary
- Standard Contractual Clauses (SCCs) — legal mechanism allowing data transfer outside EU, provided GDPR protections are contractually guaranteed by the receiving party

---

#### GCP Data Residency Enforcement
Organization Policy — Resource Location Restriction

- Uses constraint constraints/gcp.resourceLocations
- Restricts where GCP resources can be created and stored — not about data transfer
- Applied at organization, folder, or project level
- Overrides individual user permissions — even project owners cannot violate it
- Applies to Cloud SQL, BigQuery, Cloud Storage, Bigtable and many other services
- **Important**: Not enforced retroactively — existing resources must be deleted and recreated to comply
- Example for Lucy: in:eu-locations ensures no resource is created outside EU

#### VPC Service Controls

- Creates a security perimeter around GCP resources
- Prevents data from leaving the perimeter even by authorized users
- Granularity is at resource level (project, service, resource) — NOT at column or row level
- Resources inside perimeter communicate freely; traffic crossing perimeter is blocked by default
- Lucy can put patient data resources inside the perimeter and marketing data resources outside
- For column level restrictions, use BigQuery column level security or Sensitive Data Protection instead

#### Distinction between the two:

**Organization Policy** — controls where data is stored
**VPC Service Controls** — controls where data can go


#### Lucy's Clinic — GDPR Compliant Architecture on GCP
Lucy runs a holistic clinic in Paris with patients worldwide. Tom is a patient in Dallas, Texas.

**Data Residency**:

- Organization Policy restricts all resources to in:eu-locations
- VPC Service Controls prevents patient data from leaving the EU perimeter
- Patient data (Cloud SQL/Cloud Storage) stays physically in EU at all times

**Authentication & Access Control**:

- Cloud Identity — Tom logs in using federated identity (Google, corporate IdP)
- IAM with least privilege — Tom can only access his own records as a principal
- 2-factor verification — additional authentication layer for sensitive medical data
- Context-Aware Access — considers Tom's device state, IP, and location before granting access

**Solving the Latency Problem — Two-tier Architecture**:

- Static content (website, images, stylesheets, JavaScript) → cached globally via Cloud CDN → Tom gets fast website loading from a Dallas edge node
- Private patient data → single API call to EU region → retrieved directly from EU on demand
- This balances GDPR compliance with acceptable user experience for global patients

**Legal Mechanism for Cross-border Data Transfer**:

- Standard Contractual Clauses (SCCs) — alternative to strict data residency, allows data to leave EU provided GDPR protections are contractually guaranteed by the receiving party

### Site Reliability Engineering (SRE)

#### DevOps vs SRE

- DevOps is a broad culture and philosophy that breaks down the wall between development and operations teams, encouraging collaboration and continuous delivery. It does not prescribe specific rules or metrics.
- SRE is Google's opinionated and structured implementation of DevOps — it takes the DevOps philosophy and adds specific metrics, rules, practices, and roles.
- In Google's own words: "SRE is what happens when you ask a software engineer to design an operations team."


#### Who is an SRE?

- SREs are software engineers who specialize in reliability
- They have strong coding skills, infrastructure knowledge, and systems thinking
- They are not pure operations people — they can read and understand application code
- Google's rule: SREs spend 50% of their time on operations (incidents, on-call, deployments) and 50% on engineering work (automation, tooling, reliability improvements)
- The 50% is not a fixed allocation — it is a guardrail. If operations work exceeds 50%, it is a signal that the system is too unreliable and the development team needs to fix it

#### Core SRE Concepts

##### SLI — Service Level Indicator

The actual measurement of reliability
Examples: uptime percentage, response time, error rate

##### SLO — Service Level Objective

- The internal target the team agrees to
- Example: 99.9% uptime annually
- Slightly stricter than the SLA

##### SLA — Service Level Agreement

- The external contractual commitment made to customers
- Usually slightly less strict than the SLO
- Breach of SLA has contractual and financial consequences

##### Error Budget

- Derived directly from the SLO — the amount of unreliability the team is allowed to "spend"
- Example: 99.9% SLO = 0.1% downtime = approximately 8.7 hours of downtime per year
- Acts as a brake — forcing the development team to prioritize reliability over new features when the budget is exhausted
- Creates a shared incentive between development and operations
- When error budget is exhausted: freeze new feature deployments and focus on reliability improvements

##### Toil Reduction

- Toil = repetitive, manual, operational work that does not add lasting value
- Examples: manual deployments, responding to the same alerts repeatedly, manually scaling infrastructure
- SREs are expected to automate toil using their engineering skills
- Once automated, toil stays reduced with minimal maintenance
- Toil reduction frees up capacity for proactive reliability engineering

---

#### Reliability Testing Practices

##### Chaos Engineering

- Deliberately injecting failures (killing servers, dropping network connections) in a controlled and planned manner
- Goal: discover weaknesses before customers do
- Must be limited to specific zones with fallback mechanisms

- DiRT (Disaster Recovery Testing)

- Google's practice of simulating entire datacenter failures, network outages, or data corruption
- Verifies that recovery procedures actually work

##### Game Days

- Simulated major incident exercises where the entire team practices their response

##### Load Testing

- Testing system behavior under high traffic to find the breaking point before customers experience it


##### Monitoring & Alerting
- Cloud Monitoring — continuous visibility into system health via dashboards
- Cloud Logging — detailed logs for post-incident investigation
- Alerts — should be:
  - Strategic and meaningful — only fire when an SLO is at risk
  - Actionable — if the on-call engineer cannot do anything about it, it should not be an alert
  - Avoid alert fatigue — too many alerts leads to engineers ignoring them, including critical ones

- On-call — SRE engineers rotate on-call responsibility, responding to critical alerts via email, phone, or pager

---

#### Blameless Post-mortems

- Conducted after every significant production incident
- Focus is on learning, not blame — blaming individuals causes people to defend themselves rather than solve the real issue
- A formal document is produced capturing:
  - Incident timeline
  - Root cause analysis
  - Impact (users affected, duration, SLO impact)
  - Action items to prevent recurrence
  - Lessons learned
- Google shares post-mortems widely across engineering teams to build a culture of learning

### GCP CI/CD Toolchain

#### What is CI/CD?

- **CI (Continuous Integration)** — automatically builds and tests code every time a developer pushes a commit. Goal: catch issues early before they reach production
- **CD (Continuous Delivery)** — automatically deploys code that passed CI to a target environment. Goal: get validated code to production quickly and reliably
- **CI always precedes CD** — if CI fails, CD does not trigger
- Some organizations require manual approval gates before promoting to production

---

#### GCP CI/CD Toolchain Components

1. Cloud Source Repositories

- GCP's private, secure Git repository hosting service
- Alternative to GitHub, GitLab, or Bitbucket
- Keeps code within GCP for security and compliance reasons
- Cloud Build can also integrate with external repositories like GitHub, GitLab, and Bitbucket — GCP is not limited to Cloud Source Repositories

2. Cloud Build

- GCP's managed CI service — equivalent to GitHub Actions
- Triggered by code commits, pull requests, or tag creation via Cloud Build Triggers
- Configured using cloudbuild.yaml file stored in the repository
- Each build step runs inside a container ensuring:
  - Consistent environment
  - Repeatable results — eliminates "works on my machine" problem
- Responsibilities:
  - Builds source code
  - Runs unit tests
  - Runs security scanning (SAST)
  - Runs vulnerability scanning (SCA)
  - Signs container image via Binary Authorization
  - Pushes signed image to Artifact Registry
  - Triggers Cloud Deploy as the last step in cloudbuild.yaml

3. Artifact Registry

- Stores signed container images after successful Cloud Build
- Automatically scans container images for known vulnerabilities — images can become vulnerable after being built as new CVEs are discovered
- Acts as the single source of truth for deployable artifacts
- Supports code signing — publisher signs artifact with private key, downloader verifies with public key

4. Cloud Deploy

- GCP's managed CD service
- Manages progressive delivery through environments: dev → staging → production
- Deploys to GKE, Cloud Run, and GCE
- For static files (GCS), pipeline configuration defines how files are copied to buckets
- Never rebuilds — promotes the same container image that was tested in staging to production
- Supports manual approval gates between environments
- Supports rollback to previous known good artifact stored in Artifact Registry

---

#### Security in the CI/CD Pipeline

##### Binary Authorization

- GCP managed policy enforcement service
- Sits between Artifact Registry and deployment target (GKE, Cloud Run)
- Ensures only signed and verified container images are deployed
- Works with Cloud KMS for key management
- Attestor — a trusted authority that vouches a container image has passed all required checks

##### Security scanning order (before signing):

- SAST (Static Application Security Testing) — scans source code for vulnerabilities during Cloud Build
- SCA (Software Composition Analysis) — scans container image for vulnerable dependencies after build
- Container signing — only after both checks pass
- Binary Authorization verification — verifies signature before deployment


```markdown
Code push → Cloud Build triggered
     ↓
Build code → Run tests → SAST scan → SCA scan
     ↓
Sign container image (Cloud KMS)
     ↓
Push to Artifact Registry
     ↓
Cloud Build triggers Cloud Deploy (last cloudbuild.yaml step)
     ↓
Binary Authorization verifies attestation
     ↓
Cloud Deploy promotes through dev → staging → production
```
---

#### Spinnaker

- Open source, multi-cloud CD platform created at Netflix
- Still actively used in 2025, especially in multi-cloud organizations
- Integrates with Cloud Build as a CI system
- Key distinction from Cloud Deploy:
  - Cloud Deploy — GCP native, simpler, best within GCP ecosystem
  - Spinnaker — multi-cloud, more complex, better for organizations deploying across AWS, GCP, and Azure from a single platform

### GCP Observability

#### Why Observability Matters

##### Monolith vs Microservices observability challenge:

- **Monolith** — one log, one CPU graph, one memory graph — easy to observe but hard to change
- **Microservices** — multiple logs, multiple servers, multiple metrics — hard to observe but easy to change independently
- The core challenge: how do you trace a single transaction across multiple microservices?

---

#### The Three Pillars of Observability

1. **Logs — Cloud Logging**

- Centralized log aggregation from all microservices into a single view
- Searchable by Trace ID to reconstruct full transaction journey
- Acts as the central nervous system of GCP observability — other services read from it
- Supports log filter queries to find specific log entries
- Log-based metrics — save a log filter query and GCP runs it continuously in the background, generating a custom metric in Cloud Monitoring. No code required — just a filter query saved through the GCP console UI
- Example: filter logs for "retry_attempt" in refund service → creates retry counter metric automatically

2. **Traces — Cloud Trace**

- Solves the distributed transaction tracing problem in microservices
- Every request gets a unique Trace ID generated at entry point and passed through every microservice
- Each microservice operation is a Span — containing start time, end time, and duration
- A complete Trace = Trace ID + multiple Spans showing the full journey

Example trace for Tom's booking:
```markdown
Trace ID: ABC123
  ├── Span 1: API Gateway (2ms)
  ├── Span 2: Repricing Service (250ms) ← slow!
  │     ├── Span 3: Offer Store Service (200ms) ← bottleneck!
  │     └── Span 4: Pricing Service (45ms)
  └── Span 5: Tax Server (3ms)
  ```

**Two modes of operation:**

- Automatic sampling — continuously traces a small representative percentage of requests (e.g. 1-5%) in background without high cost
- On-demand tracing — for specific troubleshooting, find a specific Trace ID and examine the full detailed trace

3. **Metrics — Cloud Monitoring**

- Automatically collects metrics for GCP managed services
- Custom metrics exported via OpenTelemetry or Cloud Monitoring API
- Supports MQL (Monitoring Query Language) for custom dashboards
- Connects to log-based metrics automatically
- Supports SLO based alerts tied directly to error budget burn rate

**RED Method** — for monitoring microservices from user perspective:

R — Rate — requests per second
E — Errors — failed requests per second
D — Duration — response time per request

**USE Method** — for monitoring infrastructure:

U — Utilization — CPU, memory usage
S — Saturation — how overloaded the resource is
E — Errors — hardware or system errors


##### Additional Observability Tools
**Cloud Error Reporting**

- Works automatically in the background reading from Cloud Logging
- Groups similar errors together — 1000 occurrences of same error appear as one grouped entry
- Shows error frequency and trends — is this error increasing or decreasing?
- Notifies teams when a new error type is detected for the first time
- Links errors to specific code locations
- Saves you from manually searching logs — automatically surfaces errors and their frequency

#### Cloud Profiler

- Performance profiling tool for production environments — not a debugger
- Integrates a lightweight agent library into your application
- Runs in its own background thread — does not block application threads
- Uses statistical sampling of call stacks at regular intervals (e.g. every 10ms)
- Buffers collected samples and sends to Cloud Profiler API over HTTPS periodically
- Collects:
  - CPU profiling — which functions consume most CPU time
  - Memory profiling — which functions allocate most memory
  - Heap profiling — memory allocation and retention patterns
- Visualizes data as flame graphs showing CPU time distribution across functions
- Typical overhead: less than 1% CPU impact
- Key advantage: always on, non-invasive, historical — compare profiles over time

#### OpenTelemetry

- Open source, vendor neutral instrumentation standard fully supported by GCP
- Provides libraries for Java, Python, Go, Node.js, C++ and more
- Automatically handles Trace ID generation, Span timing, and metrics collection
- Replaces custom logger libraries with standardized instrumentation
- Data flows automatically to Cloud Trace, Cloud Logging, and Cloud Monitoring

#### How the Four Pillars Work Together

Real world example — refund service retry issue:

```markdown
Refund service logs "retry_attempt" entries
     ↓
Cloud Logging receives and stores log entries
     ↓
Log-based metric increments retry counter
     ↓
Cloud Monitoring tracks retry rate over time
     ↓
Alert fires when retry rate exceeds SLO threshold
     ↓
SRE on-call investigates
     ↓
Cloud Trace finds slow span in legacy refund service
     ↓
Cloud Logging finds exact log entries via Trace ID
     ↓
Cloud Profiler shows which function is consuming most CPU
     ↓
Cloud Error Reporting shows error frequency and code location
```

#### Connecting Observability to SRE

- RED Method metrics map directly to SLIs (error rate, latency, throughput)
- Cloud Monitoring alerts fire when SLO is at risk
- Error budget burn rate alerts trigger when reliability degrades too fast
- Log-based metrics (like retry counters) provide custom SLIs beyond standard metrics
- Blameless post-mortems use Cloud Logging, Cloud Trace, and Cloud Error Reporting to reconstruct incident timeline and root cause
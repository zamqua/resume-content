+++
title = "GCP PAC Certification Preperation Notes"
author = ["Mohammed Azam"]
date = 2026-05-26T11:05:00-05:00
tags = ["certifications", "gcppac", "notes"]
draft = false
categories: ["Tutorial"]
description: "Prep notes on GCP PAC certifcation"
showToc: true
+++

## Designing and planning a cloud solution architecture {#designing-and-planning-a-cloud-solution-architecture}


### Designing for Business Requirements {#designing-for-business-requirements}

The exam will give you a scenario with business goals and ask you to map them to
architecture decisions. Key things to internalize:

Functional vs. Non-functional requirements

Functional: what the system does (e.g., process insurance claims)
Non-functional: how it performs (availability, latency, scalability, compliance)

Workload disposition strategies — a favorite exam topic:

-   **Retain** – keep on-premises (legacy systems with no migration plan, like EHR's insurance integrations)
-   **Rehost** – lift and shift to GCP VMs
-   **Replatform** – move to managed services (e.g., MySQL → Cloud SQL)
-   **Refactor** – redesign for cloud-native (e.g., monolith → microservices on GKE)
-   **Retire** – decommission
-   **Repurchase** – replace with SaaS

Cost optimization framing:

-   CapEx (on-prem) → OpEx (cloud) shift is often a business driver
-   Committed Use Discounts (CUDs) for predictable workloads
-   Spot VMs for batch/fault-tolerant workloads

Business continuity: Always think in terms of **RTO** (how fast you recover) and **RPO**
(how much data you can afford to lose).


#### RTO &amp; RPO — The Two Pillars of Disaster Recovery {#rto-and-rpo-the-two-pillars-of-disaster-recovery}

-   RTO (_how long can system be down?_)
    -   Recovery Time Objectives. It's measured forward in time from the
        disaster.
    -   If RTO is 1 hour and system is down at 2:00 PM then the system should
        recover by 3:00 PM. Company can only effort to lose 1 HOUR of data.
-   RPO (_how much data can you afford to lose?_)
    -   Recovery Point Objectives. It's measured backwards in time from the
        disaster.
    -   If RPO is 4 hours then you can tolerant losing up to 4 hours of data.
    -   If a company server is down at 2 PM and their last backup ran at 10 AM
        then they lose 4 hours of data. It is up to their target of 4 HOURS. But
        this is unacceptable for EHR. The RPO is subjective to the functional
        domain.
-   How RPO and RTO decided?
    -   It is business decisions.
    -   Depends on the data and domain? For example it is very critical losing
        patient data and hence its RPO/RTO will be near-zero.
-   Key Relationship
    Lower RPO/RTO = More expensive, more complex
    Higher RPO/RTO = Less expensive, but more risk

| RPO/RTO   | TargetGCP Solution                              | Cost   |
|-----------|-------------------------------------------------|--------|
| Hours     | Scheduled snapshots + Cloud Storage backups     | Low    |
| Minutes   | Cloud SQL with read replicas, Filestore backups | Medium |
| Near-zero | Cloud Spanner (multi-region), Cloud SQL HA      | High   |


### Designing for Technical Requirements {#designing-for-technical-requirements}

High Availability (HA) patterns:

Multi-region deployments for global users
Regional for most enterprise workloads (99.99% SLA on many GCP services)
Zonal for dev/test or cost-sensitive workloads
GKE with regional clusters = automatic multi-zone node distribution

Scalability:

Horizontal scaling preferred on GCP (more instances, not bigger ones)
GKE Autopilot and Cloud Run scale to zero
Managed instance groups (MIGs) with autoscaling for VMs

The Google Cloud Well-Architected Framework pillars — know these cold:

Operational Excellence
Security
Reliability
Performance Optimization
Cost Optimization
Sustainability


### Designing Network, Storage, and Compute Resources {#designing-network-storage-and-compute-resources}

Networking essentials:

VPC – global, spans regions; subnets are regional
Shared VPC – one host project, multiple service projects share networking (good for large orgs)
VPC Peering – connect two VPCs (no transitive routing)
Private Service Connect – access Google APIs/services privately without public IPs
Cloud Interconnect – dedicated high-performance on-prem to GCP (EHR's use case for "secure, high-performance connection")
Cloud VPN – encrypted tunnel over internet, lower cost than Interconnect

Storage selection — the exam loves this:
NeedServiceRelational, global scaleCloud SpannerRelational, regionalCloud SQLNoSQL, documentFirestoreNoSQL, wide-column, high throughputBigtableObject storageCloud StorageIn-memory cacheMemorystore (Redis)Data warehouseBigQuery
Compute selection:

Compute Engine – full VM control, lift-and-shift
GKE – containerized workloads, complex microservices
Cloud Run – stateless containers, event-driven, scale to zero
Cloud Run Functions – single-function event triggers
App Engine – fully managed app platform (less common in new designs)


### Migration Planning {#migration-planning}

Key tool: Google Cloud Migration Center — discovers, assesses, and plans migrations.
Migration phases to know:

Assess – inventory workloads, dependencies, TCO
Plan – design landing zone, network topology
Deploy – migrate in waves
Optimize – right-size, modernize

Important exam trap: When a case study says legacy systems are staying on-prem (like EHR's insurance integrations), the answer will NOT involve migrating those — it will involve hybrid connectivity to reach them.

1.5 Envisioning Future Improvements
The exam may ask about cloud-first design principles or evolving an architecture over time. Think:

Replacing VMs with containers → serverless over time
Moving batch pipelines to streaming (Pub/Sub + Dataflow)
Incorporating gen AI/ML capabilities (Vertex AI, Agent Builder)

What the Exam Specifically Tests in Domain 1

Choosing between Cloud Interconnect vs. VPN given requirements
Picking the right database for a given workload
Identifying which workloads to migrate vs. keep on-prem
Translating "99.9% availability" into architecture decisions (multi-zone vs. multi-region)
Recognizing CapEx/OpEx trade-offs in business scenarios


### Quick Practice Questions {#quick-practice-questions}

Give these a shot before we move on:
Q1. EHR Healthcare needs a secure, high-bandwidth, low-latency connection between their on-premises data center and GCP. Which service should you recommend?
Q2. A company needs a globally consistent relational database that can serve users in North America, Europe, and Asia with strong consistency. Which GCP service is the best fit?
Q3. Your team is migrating a legacy application that cannot be modified. The priority is speed of migration, not optimization. Which workload disposition strategy applies?

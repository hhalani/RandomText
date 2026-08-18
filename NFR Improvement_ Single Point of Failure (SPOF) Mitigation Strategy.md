# NFR Improvement: Single Point of Failure (SPOF) Mitigation Strategy

## 1. Purpose

The objective of this assessment is to identify and mitigate potential Single Points of Failure (SPOFs) within the current architecture and improve overall system availability and resiliency.

The current architecture consists of Experience APIs interacting directly with downstream systems such as Guidewire PolicyCenter, other enterprise services, AWS infrastructure, on-premises systems, and vendor SaaS platforms.

The recommendations in this document are focused on **improving NFRs within the existing architecture**. They do not require the adoption of the future-state layered architecture, which is being addressed separately.

## 2. What is a Single Point of Failure?

A Single Point of Failure is a component, dependency, infrastructure element, or integration path where a failure can cause disruption or unavailability of a business service.

SPOFs can exist at multiple levels:

- Application / service instances
- Database
- Messaging infrastructure
- Network connectivity
- External SaaS dependencies
- Authentication and authorization
- Cache
- Load balancer / API gateway
- Deployment infrastructure
- Cloud availability zone or region

The goal is not necessarily to eliminate every possible failure, but to ensure that the failure of an individual component does not result in an unnecessary outage or cascading failure across the broader system.

## 3. Key SPOF Areas and Recommended Improvements

### 3.1 Application / Service Layer

Experience APIs and other Spring Boot services should be deployed as multiple stateless instances rather than relying on a single running instance.

**Recommendations:**

- Deploy multiple instances of each service.
- Distribute instances across multiple Availability Zones where possible.
- Use load balancing to distribute traffic.
- Implement health checks and readiness probes.
- Automatically remove unhealthy instances from service.
- Ensure services are stateless and do not depend on local instance state.
- Use auto-scaling to handle demand and instance failures.

**Target State:**

```text
                    Load Balancer
                         |
              +----------+----------+
              |                     |
          Service A              Service A
             AZ-1                   AZ-2
              |                     |
              +----------+----------+
                         |
                    Downstream
                    Services
```

Failure of one service instance should not result in service unavailability.

---

### 3.2 Database

A single database instance can become a critical SPOF even when the application tier is highly available.

**Recommendations:**

- Use database clustering or managed Multi-AZ capabilities.
- Enable automatic failover where supported.
- Implement database replication.
- Monitor database health, capacity, and replication lag.
- Avoid application designs that depend on a single database node.

The application HA strategy should be aligned with the availability characteristics of the underlying database.

---

### 3.3 Messaging Infrastructure

Messaging platforms such as Kafka, Solace, SQS, and other brokers should also be evaluated for SPOFs.

**Recommendations:**

- Use clustered/replicated brokers.
- Distribute brokers across Availability Zones where supported.
- Configure appropriate replication factors.
- Use consumer groups for high availability.
- Implement Dead Letter Queues/Topics.
- Ensure messages can be replayed or recovered after transient failures.
- Monitor broker health and consumer lag.

The messaging platform should not become a single dependency whose failure brings down multiple services.

---

### 3.4 External SaaS and Guidewire Dependencies

Guidewire PolicyCenter and other vendor SaaS platforms represent **external managed dependencies**.

The underlying infrastructure HA is primarily managed by the vendor; therefore, the consuming applications should focus on **failure isolation and graceful handling of downstream failures**.

For example:

```text
Experience API
      |
      v
+-----------------------+
| Resiliency Controls   |
|-----------------------|
| Timeout               |
| Retry + Backoff       |
| Circuit Breaker       |
| Bulkhead              |
| Rate Limiting         |
+-----------+-----------+
            |
            v
     Guidewire / SaaS
```

**Recommendations:**

- Configure appropriate connection and response timeouts.
- Implement controlled retries with exponential backoff and jitter.
- Implement circuit breakers.
- Use bulkheads to prevent one failing dependency from consuming all application resources.
- Implement rate limiting where appropriate.
- Support graceful degradation where business requirements permit.
- Consider asynchronous processing for business processes that do not require an immediate downstream response.
- Monitor downstream dependency availability and latency.

The objective is to prevent a downstream SaaS outage from cascading into an outage of the Experience API or other dependent services.

---

### 3.5 Connection Pool and Resource Exhaustion

Connection pools can become an indirect SPOF.

For example, if a downstream system becomes slow, all HTTP connections may remain occupied. Eventually, the application's connection pool can become exhausted, causing otherwise healthy requests to fail.

**Recommendations:**

- Configure connection timeouts.
- Configure read/response timeouts.
- Establish appropriate maximum connection limits.
- Use bulkhead isolation.
- Monitor connection pool utilization.
- Prevent unlimited request queuing.
- Establish dependency-specific resource limits where appropriate.

---

### 3.6 Cache

If a cache such as Redis/ElastiCache becomes unavailable, the impact depends on whether the application requires the cache to operate.

**Recommendations:**

- Use replicated/Multi-AZ cache deployment where supported.
- Enable automatic failover.
- Avoid making the cache a mandatory dependency unless required.
- Design applications to degrade gracefully when the cache is unavailable.
- Monitor cache availability, memory utilization, and connection health.

A cache should ideally improve performance rather than become a hard dependency for basic application availability.

---

### 3.7 AWS and Cloud Infrastructure

AWS infrastructure should be evaluated for dependencies that exist within a single Availability Zone or region.

**Recommendations:**

- Deploy critical services across multiple Availability Zones.
- Use managed AWS services with built-in HA where appropriate.
- Avoid single-instance infrastructure for critical workloads.
- Use Auto Scaling where appropriate.
- Evaluate multi-region architecture for business-critical applications where the availability requirement justifies the additional complexity.

Multi-region deployment should be considered based on business RTO/RPO requirements rather than being automatically applied to every service.

---

### 3.8 AWS ↔ On-Premises Connectivity

Hybrid connectivity can become a significant SPOF.

For example:

```text
AWS
 |
Single Network Connection
 |
On-Prem
```

Failure of the network connection can make otherwise healthy applications unavailable.

**Recommendations:**

- Use redundant connectivity paths.
- Consider multiple Direct Connect connections where appropriate.
- Use VPN as a backup path where appropriate.
- Ensure network devices and firewalls have redundancy.
- Monitor connectivity and failover.
- Define recovery procedures for network failures.

---

### 3.9 DNS, Load Balancer and API Gateway

Infrastructure components such as DNS, load balancers, API gateways, and service discovery mechanisms should also be included in the SPOF assessment.

**Recommendations:**

- Use highly available managed services where possible.
- Avoid single-instance gateways or ingress components.
- Ensure health checks are configured correctly.
- Implement appropriate failover mechanisms.
- Monitor availability and latency.

---

### 3.10 Deployment and Release Process

A service can also experience a SPOF during deployment if all instances are replaced or restarted simultaneously.

**Recommendations:**

- Use rolling deployments.
- Consider blue/green deployments for critical services.
- Use canary deployments where appropriate.
- Support automated rollback.
- Maintain backward compatibility between service versions.
- Avoid deployments that require simultaneous downtime of all instances.

A failed deployment should affect the minimum possible number of users and services.

---

## 4. Recommended Resiliency Capabilities for Spring Boot Services

A common resiliency capability should be established across Experience APIs and other applicable Spring Boot services.

The following capabilities should be standardized:

| Capability | Purpose |
|---|---|
| Timeout | Prevent indefinite waiting on downstream systems |
| Retry | Handle transient failures |
| Exponential Backoff + Jitter | Prevent retry storms |
| Circuit Breaker | Stop calls to an unhealthy dependency |
| Bulkhead | Prevent one dependency from exhausting service resources |
| Rate Limiting | Protect services and downstream systems |
| Fallback / Graceful Degradation | Maintain partial functionality when possible |
| Health Checks | Detect unhealthy instances |
| Observability | Identify and troubleshoot failures |
| Correlation ID | Trace requests across services |
| Alerting | Detect failures before they become widespread |

These capabilities can be implemented within the **current architecture** without requiring a transition to the future layered architecture.

## 5. Recommended Approach

Rather than attempting to eliminate every possible SPOF immediately, the recommended approach is to prioritize SPOFs based on:

1. Business criticality
2. Probability of failure
3. Impact of failure
4. Existing recovery mechanism
5. Recovery Time Objective (RTO)
6. Recovery Point Objective (RPO)
7. Cost and complexity of remediation

The initial focus should be on dependencies that can create **cascading failures**, particularly downstream service failures, connection pool exhaustion, database availability, messaging infrastructure, and network connectivity.

## 6. Proposed Target Outcome

The desired outcome is an architecture where:

> **Failure of an individual service instance, infrastructure component, network path, or downstream dependency does not unnecessarily result in an end-to-end business service outage.**

The SPOF remediation should therefore focus on both **High Availability (HA)** and **Failure Isolation**.

HA ensures that alternate instances or infrastructure are available when a component fails.

Failure isolation ensures that when an unavoidable dependency failure occurs—such as a Guidewire or SaaS outage—the failure is contained and does not cascade throughout the application ecosystem.

## 7. Implementation Priority

### Priority 1 – Immediate

- Identify critical SPOFs.
- Implement timeout standards.
- Implement circuit breakers.
- Implement controlled retries.
- Implement bulkheads for critical downstream dependencies.
- Ensure critical services have multiple running instances.
- Establish health checks and monitoring.

### Priority 2 – Near Term

- Review database HA.
- Review messaging HA.
- Review AWS Multi-AZ deployment.
- Review AWS/on-prem network redundancy.
- Establish standardized resiliency patterns across services.
- Improve observability and alerting.

### Priority 3 – Strategic

- Evaluate multi-region requirements for critical workloads.
- Evaluate asynchronous processing for appropriate business processes.
- Establish enterprise-wide resiliency standards.
- Align SPOF remediation with the future-state layered architecture.

## 8. Key Principle

The immediate objective is **not to redesign the current architecture**. The objective is to improve its NFR characteristics and reduce operational risk while the future-state layered architecture is being developed separately.

This allows the organization to obtain near-term resiliency improvements without waiting for the broader architectural transformation.
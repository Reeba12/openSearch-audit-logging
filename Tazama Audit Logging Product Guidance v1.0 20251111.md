# Tazama Audit Logging Product Guidance v1.0

**Author:** Sandy Labuschagne - Tazama
**Date:** November 2025

## Table of Contents

1. [Introduction](#1-introduction)
   1. [Tazama System Overview](#11-tazama-system-overview)
   2. [Tazama Ecosystem Overview](#12-tazama-ecosystem-overview)
   3. [Tazama Logging Overview](#13-tazama-logging-overview)
   4. [Document Purpose](#14-document-purpose)
2. [Audit Logging Overview](#2-audit-logging-overview)
   1. [Audit Logging Purpose](#21-audit-logging-purpose)
   2. [Audit Logging vs System Logging](#22-audit-logging-vs-system-logging)
3. [Current Tazama System Logging Pattern](#3-current-tazama-system-logging-pattern)
   1. [Processor](#31-processor)
   2. [Event-Sidecar](#32-event-sidecar)
   3. [NATS Messaging System](#33-nats-messaging-system)
   4. [Lumberjack](#34-lumberjack)
   5. [ELK Stack (Elasticsearch, Logstash, Kibana)](#35-elk-stack-elasticsearch-logstash-kibana)
   6. [Flow Overview](#36-flow-overview)
   7. [Key Points](#37-key-points)
4. [Principles for Audit Logging Sub-system](#4-principles-for-audit-logging-sub-system)
5. [Assumptions](#5-assumptions)
6. [Tazama Audit Logging Architecture](#6-tazama-audit-logging-architecture)
   1. [Proposed Solution](#61-proposed-solution)
   2. [Data Flow](#62-data-flow)
   3. [Considerations for Synchronous Logging](#63-considerations-for-synchronous-logging)
7. [Audit Logs & Data Lakehouse Integration](#7-audit-logs--data-lakehouse-integration)
   1. [Benefits of This Architecture](#71-benefits-of-this-architecture)
   2. [Implementation Patterns](#72-implementation-patterns)
8. [Appendix: Audit Logging Architecture Research](#8-appendix-audit-logging-architecture-research)
   1. [Apache 2.0 Solution Options Investigated](#81-apache-20-solution-options-investigated)
   2. [Log Collection Alternatives](#82-log-collection-alternatives)
   3. [NATS vs Data Prepper](#83-nats-vs-data-prepper)
   4. [Audit Logging Architecture Comparison](#84-audit-logging-architecture-comparison)

---

## 1 Introduction

### 1.1 Tazama System Overview

The Tazama Transaction Monitoring System is a rules-based forward-chaining inference engine that ingests transaction data as JSON-formatted ISO20022 messages and evaluates the data in real-time. Tazama uses a number of pre-built and configured rule processors to look for fraud and money-laundering behaviors. Rule results are summarized into fraud and money-laundering scenarios (known as typologies) and the system is able to block or flag transactions if typologies breach configured thresholds.

The core detection capability within the platform is distributed within the steps in the end-to-end evaluation flow below.

Once data is ingested into the transaction history by the TMS API, the Event Director (ED) performs an initial "triage" step to determine if the transaction should be inspected by the platform, and in what way. Currently this is a very simple decision based on the transaction type only (i.e. pain.001, pain.013, pacs.008 and pacs.002), though we envisage that the decision-making here can be more complex in the future by inspecting attributes contained in the message. For now, the ED uses the transaction type to select the typologies that are to be evaluated and triggers the rules required by the typologies. The ED routing is configured via a network map that defines the hierarchy of typologies and rules.

Each rule processor that receives the trigger payload from the Event Director evaluates the transaction and the historical behavior of the transaction participants according to the rule's specification and configuration. Rule processors are driven by a combination of parameters and result category specifications to determine only one of a configurable set of related outcomes. The rule outcome is then submitted to the typology processor for scoring.

The typology processor assigns a weighting to each rule outcome as it is received based on the rule's parent typologies' configurations. Once all the rule results for a specific typology have been received, the typology adds all the weighted scores together to form the typology score. The typology score can be evaluated against an "interdiction" threshold to determine if the client system should be instructed to block a transaction "in flight". The typology configuration also contains an investigation threshold that will trigger a review or investigation process at the end of the transaction evaluation.

Once these three steps are complete, the evaluation of the transaction is wrapped up in the Transaction Aggregation and Decisioning Processor where the results from typologies are combined and reviewed to determine if an investigation alert should be sent to the Case Management System. If any typology had breached either its investigation or interdiction threshold, the system would trigger an investigation alert for that transaction.

The Tazama system is currently provided as a software engine and as such is generally implemented as a back-end service that taps into transaction flows.

### 1.2 Tazama Ecosystem Overview

The Tazama Transaction Monitoring Service (TMS) is intended as a core component of a wider ecosystem that comprises of several components designed to combine into a comprehensive fraud and money-laundering management ecosystem.

Numerous features in the ecosystem are currently under development and can be summarized as follows:

#### 1.2.1 Tazama Connection Studio

This component is designed to accelerate the deployment of Tazama into a new environment by allowing an implementer to rapidly create and implement REST API-based interfaces and end-to-end data pipelines in the Tazama TMS.

#### 1.2.2 Relay Service Enhancement

The Relay Service component in the Tazama ecosystem is a light-weight point-to-point router for message traffic generated by Tazama into downstream systems, such as the Case Management System and upstream systems, such as the client transaction system. The Relay Service is designed to be highly modular and extensible and allows for multiple output interfaces that can be selected at deployment, with an easy-to-use integration pattern for adding additional target interfaces.

#### 1.2.3 Case & Investigation Management System

Every alert that is generated out of the Tazama TMS must be investigated to either confirm or refute the suspicious behavior identified by the TMS. This is firstly required to maintain the good health of the monitoring service itself, but if suspicious behavior is confirmed, the operator of the TMS also has an obligation to investigate the alert for fraud and money-laundering regulatory and legislative enforcement purposes.

The Case and Investigation Management System (CIMS) provides the processes and the tools to assist investigators in the fulfilment of this obligation.

The CIMS also contains an AI-powered Alert Triage Module that tries to automatically validate, classify and prioritize the alerts generated out of the TMS before an investigator is assigned to minimize unnecessary investigative effort.

#### 1.2.4 Business Intelligence, Analytics and Reporting

The Tazama Business Intelligence, Analytics and Reporting (BIAR) component provides the reporting and advanced modeling capabilities as input into model management and model development, as well as standard reporting for overall operational management of the ecosystem and regulatory reporting to fulfil regulatory obligations.

**Data Lakehouse**

Tazama will implement a Data Lakehouse architecture rather than a traditional data warehouse or data lake. This design combines the scalability and flexibility of data lakes with the transactional reliability and performance of data warehouses—providing a unified foundation that supports real-time analytics, machine learning, and BI use cases.

The Data Lakehouse will ingest data from the following sources:

1. Transaction Monitoring System (TMS) ODS
2. Case and Investigation Management System (CIMS) ODS
3. Metadata from CIMS Unstructured Documents
4. External Data Sources

The Data Lakehouse will serve as a central repository, providing curated data and data products to:

1. The Investigation Module for advanced analytics and visualizations
2. The Simulation Sandbox for rule evaluation and historical replay
3. Calibration Module for typology calibration and optimization
4. Anomaly detection for new rule identification
5. Tazama rule studio for new rule parameterization and development

**BIAR module**

The BIAR module will be built around JupyterLab, a leading open-source environment for data science, artificial intelligence, and machine learning. It enables flexible, notebook-driven exploration, advanced analytics, and model development by technical users.

To support broader, non-technical audiences, a self-service Business Intelligence (BI) tool—Apache Superset—will be deployed alongside JupyterLab. Superset enables drag-and-drop dashboarding, ad hoc exploration, and report sharing through a secure, role-based interface.

The BIAR ecosystem will support:

1. Operational monitoring – dashboards and KPIs for real-time oversight
2. Regulatory reporting – scheduled and auditable reports for compliance
3. Data exploration and feature engineering – via notebooks and datasets
4. Model development and scoring – using integrated Machine Learning (ML) tooling
5. Collaboration and knowledge sharing – through reusable, curated data products

#### 1.2.5 Model Management

The Tazama Model Management component, containing a configuration utility and a low-code rule-builder (Tazama Rule Studio), provides accessible tools to an operator to add new rules and typologies and fine-tune their existing rules and typologies.

#### 1.2.6 Simulation Sandbox

The Business Intelligence, Analytics and Reporting module, and the Model Management module, are supported by a simulation sandbox where new models, rules and typologies can be tested before they are deployed into a production system.

### 1.3 Tazama Logging Overview

Currently in the Tazama system, a system logging sub-system has been implemented, but there is no application level audit logging functionality. Audit logging has been specified as a key requirement in a number of new components being developed such as CIMS, Rule Studio and Tazama Connection Studio. Audit logging is also a requirement in the core Tazama system for services such as the TMS API and the admin-service.

### 1.4 Document Purpose

The purpose of this document is to provide an overview of guidelines and requirements for audit logging to aid in the design and development of this sub-system in the Tazama ecosystem.

---

## 2 Audit Logging Overview

### 2.1 Audit Logging Purpose

Audit logging is a far more strict and rigid form of logging and is required in systems where security, transparency, veracity and accountability are paramount.

The purpose of audit logging is:

1. To support investigations into activities that have occurred within the system boundary
2. To provide data in support of ascertaining the chain of events leading up to an event of interest with a view to identifying the actors and the actions they undertook which resulted in a particular outcome

#### 2.1.1 Key Properties of Audit Logs

- **Non-repudiation:** Users cannot deny having performed the action
- **Immutability:** Logs must not be alterable once written
- **Completeness:** Must capture all critical actions (e.g. logins, data updates, permission changes)
- **Accountability:** Each action must be traceable to a specific actor (user or system)

#### 2.1.2 Examples of Events Captured in Audit Logs

- User login/logout (success and failure)
- Creation, modification, or deletion of records (e.g. cases, tasks, messages)
- Access to sensitive data (view actions, not just changes)
- Changes to user permissions or roles
- Manual overrides
- System configuration changes

### 2.2 Audit Logging vs System Logging

| Aspect | System Logging | Audit Logging |
|---|---|---|
| Purpose | Help developers/operators understand system behavior and debug issues | Provide a record of user actions and security-relevant events for accountability, compliance, and forensics |
| Audience | Developers, DevOps, SREs | Auditors, Compliance Officers, Security Teams |
| Focus | System state, errors, function flows, performance | Who did what, when, and where — focused on user actions and data changes |
| Typical Content | Stack traces, variable values, service calls, errors | Actor ID, action type, target resource, timestamp, result/outcome |
| Log Levels | Uses levels like DEBUG, INFO, WARN, ERROR, FATAL | Generally no log levels — every event is logged consistently |
| Structure | Often unstructured or semi-structured text; may use JSON | Strongly structured, often JSON or database record with strict fields |
| Tamper Protection | Not always required | Should be immutable or tamper-evident (e.g. WORM storage, cryptographic signing, append-only DB) |
| Retention | Retention policy based on operational needs | Often mandated by regulation (e.g. 5-7 years) |

---

## 3 Current Tazama System Logging Pattern

System logging in Tazama is performed through a modular stack that centers around high-performance, structured log collection and delivery using several coordinated components:

### 3.1 Processor

- The main Node.js application that generates log events using the `LoggerService` (from `@tazama-lf/frms-coe-lib`)
- Supports multiple log levels (`trace`, `debug`, `log`, `warn`, `error`, `fatal`)
- Log level is controlled by an environment variable that each microservice uses to determine the log level
- Logs are encoded using protobuf and has a dedicated proto file in the frms-coe-lib called Lumberjack.proto
- Sends logs to an event-sidecar via gRPC (in dev/test runtime environment the logs will only be pushed to console(stdout) and not use gRPC)

### 3.2 Event-Sidecar

- A microservice running alongside each processor
- Acts as a gRPC server, receiving log messages from the processor
- Formats logs and publishes them to a NATS subject/topic

### 3.3 NATS Messaging System

- Used for inter-process communication between the event-sidecar and lumberjack
- Event-sidecar publishes log messages; lumberjack subscribes to receive them

### 3.4 Lumberjack

- Microservice that receives log messages from NATS
- Batches log events before sending them to Elasticsearch (reducing I/O and improving performance)
- Configuration options determine batching behavior (e.g. `FLUSHBYTES`)

### 3.5 ELK Stack (Elasticsearch, Logstash, Kibana)

- Centralized logging and analysis
- Lumberjack transmits batched logs to Elasticsearch for storage and visualization

### 3.6 Flow Overview

1. Processor emits a log event
2. Sends the log via gRPC to the event-sidecar
3. Event-sidecar publishes the log to a NATS subject
4. Lumberjack receives logs from NATS, batches them, and sends to Elasticsearch.
5. Logs are visualized and analyzed in Kibana

### 3.7 Key Points

- All configuration (such as log level, NATS server/subject, batching size) is managed via environment variables in each service
- The logger supports context-rich logging with optional service operation names, identifiers, and callbacks
- The stack is designed for scalability, performance, and reliability, supporting high-throughput logging with minimal resource overhead
- Elastic APM integration is available for monitoring application performance

---

## 4 Principles for Audit Logging Sub-system

| # | Sub-category | Principle/Guidance |
|---|---|---|
| P1 | Deployment | Audit logging components must be able to be deployed in the cloud or on premises in containers |
| P2 | Licensing | Audit logging components must be licensed under Apache 2.0 or to the left i.e. including MIT |
| P3 | Access control | Only authorized personnel may view or search audit logs, using role-based access controls (RBAC). RBAC must be via the Tazama authentication service (default IAM Keycloak). |
| P4 | Multi-tenancy | Audit logging must be differentiated by Tenant (identified by tenant id) |
| P5 | Atomicity | An atomicity principle must apply to audit logging meaning that an operation is indivisible and all-or-nothing. Either the entire operation (writing the audit log and completing the underlying event) completes successfully, or none of it happens at all. There is no partial completion or in-between state. |
| P6 | Audit logging failure | If audit logging functionality is disabled (or fails) for whatever reason, the functionality being audit logged should be blocked to ensure no auditable function is carried out while audit functionality is disabled |
| P7 | Performance | Audit logging should not compete with operational components for infrastructure |
| P8 | Audit Trail Integrity | Audits must be tamper-evident and not tamper-proof. [SL1.1][SL1.2] Audit trails should include a mechanism such as hashing or digital signatures [SL2.1][SL2.2] to detect log tampering |
| P9 | Anomaly detection | Support for anomaly detection (e.g. sudden spikes in login) will be done by ingesting audit data into the Tazama TMS API and evaluation by the core Tazama system |
| P10 | Audit log content | The system must differentiate between system-driven actions and human actions |
| P11 | Real-time | Audit logs should be generated and stored in real time |
| P12 | Scalability | The audit logging system must support high write throughput environments |
| P13 | Time Synchronization | All audit log timestamps must be based on a reliable time source (e.g. NTP) to ensure consistency |
| P14 | Data Captured | Each audit log entry must include: Timestamp (ISO 8601 format, UTC); User ID(s) or system process ID(s) of relevant actor; Event type and description; Affected resource(s) (e.g. account ID, transaction ID); Source IP address and/or device ID/VPN; Application or service name; Result status (e.g. SUCCESS, FAILURE); Status change (if change of state is involved for an object); List of "From value" (value before change) and After value (value post change) of all affected fields, in json format; TenantID the logged in actor is behaving as |
| P15 | Standards | Auditing must comply with relevant standards and regulations, such as: PCI-DSS; SOX; GDPR (if personal data is logged); ISO 27001 |
| P16 | Reporting and Analytics | Reporting and Analytics on audit logs will be via the BIAR module (in the longer term) |
| P17 | Lakehouse Integration | Audit Log / Data Lakehouse Integration Pattern should be batch, with a configurable sync schedule e.g. hourly |

---

## 5 Assumptions

| # | Sub-category | Assumption |
|---|---|---|
| A1 | What to log | Detailed audit logging requirements will be specified in the Business Requirement Specification (BRS) documents for each Tazama component |
| A2 | Encryption | The decision on encryption at rest and encryption in transit for audit logging will be deferred to the encryption decisions for the whole Tazama platform |
| A3 | What to log | Audit logging will not log the handoff between inter-service processors, but logging of user actions and inter-system actions |
| A4 | What to log | Only information that is available in the system can be logged e.g. if device id is not obtained it can't be logged. This may give rise to new requirements to source and ingest additional data for audit logging purposes |
| A5 | What to log | Audit logging will only create audit logs for functionality where Tazama has direct control over the development i.e. excluding external components such as AIM (Keycloak). If AIM audit logs are required for reporting or analytics purposes, those logs will need to be made available separately |
| A6 | What to log | Transactions evaluated in Tazama will not be records as an audit logging event. The outcome of the transaction evaluation is in the evaluation |
| A7 | How to log | All audit log writes to the database must be implemented using asynchronous programming patterns (promise/await or equivalent) that block execution until write confirmation is received. The application code must not proceed with subsequent operations until the audit log entry has been successfully committed to the database. |

---

## 6 Tazama Audit Logging Architecture

### 6.1 Proposed Solution

The proposed solution for audit logging in Tazama is a synchronous real-time application audit logging architecture using OpenSearch involving direct, immediate data transfer from the application to the OpenSearch Cluster.

### 6.2 Data Flow

- **Event Generation:** The real-time application generates an audit event for critical action (e.g. user login attempt, data access, configuration change).
- **Synchronous Ingestion Request:** The application makes a direct, blocking API call (usually HTTP POST with a JSON payload) to the OpenSearch cluster endpoint or OpenSearch Ingestion endpoint to log the event.
- **Processing and Indexing:** OpenSearch receives the event, processes it, and indexes it into the appropriate audit log index. This operation is completed before the application proceeds.
- **Acknowledgment and Continuation:** OpenSearch sends an acknowledgment back to the application. Upon receiving successful acknowledgment, the application continues its main execution flow.
- **Analysis and Monitoring:** Security teams and administrators use OpenSearch Dashboards to view, search, and analyze audit logs in near real-time, potentially configuring alerts for suspicious activities.

### 6.3 Considerations for Synchronous Logging

- **Performance Impact:** Synchronous logging adds latency to application operations because the application waits for the logging operation to be completed. This approach is typically reserved for critical, non-repudiable audit events where an unlogged event would be a compliance violation.
- **Availability:** The application's availability becomes dependent on the OpenSearch cluster's availability. Robust error handling and potential asynchronous fallbacks are crucial to prevent application outages if OpenSearch is unreachable.
- **Throughput and Scalability:** The OpenSearch cluster must be adequately provisioned to handle the peak synchronous write load to avoid becoming a bottleneck. Utilizing dedicated master nodes and scaling data nodes horizontally helps maintain performance.
- **Data Consistency:** This architecture provides strong consistency guarantees for audit logs, ensuring that if an action is performed, it is logged immediately and reliably.

---

## 7 Audit Logs & Data Lakehouse Integration

The following architecture flow is proposed as a future state for ingesting audit log data into the Data Lakehouse for reporting and analytical purposes. In the short term, reporting and analytical requirements for audit logs will be fulfilled by OpenSearch tools or by connecting to the OpenSearch database from Analytical and Reporting tools.

### 7.1 Benefits of This Architecture

#### 7.1.1 Operational

- Long-term retention at lower cost than OpenSearch
- Historical analysis beyond OpenSearch retention limits
- Compliance with data governance requirements

#### 7.1.2 Analytics

- Complex analytics with SQL and ML frameworks
- Cross-system correlation with business data
- Advanced visualizations and reporting

#### 7.1.3 Cost Optimization

- Tiered storage: Hot (OpenSearch) + Cold (Data Lake)
- Compression and efficient file formats
- Compute-storage separation

### 7.2 Implementation Patterns

#### 7.2.1 Batch Export Approach

- Tools: Logstash, custom scripts, OpenSearch APIs
- Frequency: Hourly, daily, or based on index rotation
- Format: JSON

#### 7.2.2 Streaming Replication

- Tools: Fluent Bit, Kafka, or NATS with dual outputs
- Benefit: Same data flows to both systems simultaneously

#### 7.2.3 CDC-Style Approach

- Tools: Custom exporters, OpenSearch snapshots
- Use case: When you need to capture changes/updates to logs

---

## 8 Appendix: Audit Logging Architecture Research

This section includes the background research that was conducted into audit logging architecture (mainly for asynchronous logging alternatives) and is entirely optional to read.

### 8.1 Apache 2.0 Solution Options Investigated

The following Apache 2.0 open source components were investigated as the solution stack for audit logging in Tazama:

- **Collection:** Fluent Bit [SL3.1]
- **Storage/Search:** OpenSearch
- **Visualization:** OpenSearch Dashboards

Combining Fluent Bit with OpenSearch for audit logs creates a robust and efficient system for collecting, processing, and analyzing security-related events.

Fluent Bit is an extremely efficient log processor that allows us to collect logs from various sources, prepare them, and forward them to one or more destinations.

Data Prepper is part of the OpenSearch eco-system and is a data collection and transformation tool specifically designed to ingest, process, and route observability data (logs, metrics, traces) from various sources into OpenSearch with built-in processors for parsing, filtering, and enrichment.

OpenSearch is a fork of Elasticsearch and Kibana. The OpenSearch application is a search engine that stores data in JSON format. For visualization, OpenSearch dashboards can be used.

#### 8.1.1 Strengths for Audit Logging

The proposed solution architecture has the following strengths:

- **Scalability & Performance:** OpenSearch provides the distributed architecture and search performance needed for high-volume audit data across enterprise environments.
- **Data Integrity:** OpenSearch offers index-level controls, immutable storage options, and snapshot capabilities that support audit trail preservation requirements.
- **Flexible Ingestion:** Fluentd/Fluent Bit can collect audit logs from diverse sources (applications, databases, systems, security tools) with reliable delivery guarantees.
- **Rich Search & Analytics:** Complex audit queries, correlation across systems, and compliance reporting are well-supported through OpenSearch's query capabilities.
- **Retention Management:** Automated index lifecycle management, archival, and deletion policies align with regulatory retention requirements.

#### 8.1.2 Gaps to Address for Full Audit Compliance

- **Cryptographic Integrity:** The base stack doesn't include built-in log signing or hash chaining. This would need to be implemented at the application level or through custom plugins.
- **Access Controls:** While OpenSearch has role-based access control, you may need additional layers for strict audit log segregation and non-repudiation.
- **Immutability Guarantees:** Additional controls like write-once storage backends or external integrity validation systems may be required for highly regulated environments.
- **Time Synchronization:** Ensure all log sources use NTP for reliable timestamps - this is infrastructure-level, not specific to the logging stack.
- **Multi-tenancy:** How can we ensure anyone accessing the OpenSearch dashboards are only able to see the logs of the applicable tenant?

#### 8.1.3 Enhanced Compliance Approaches

- **Application-Level Signing:** Implement cryptographic signing in applications before logs reach the collection layer.
- **Dedicated Audit Indices:** Use separate OpenSearch indices with stricter access controls and retention policies for audit data.
- **External Validation:** Implement periodic cryptographic validation of stored audit logs through separate systems.
- **Compliance Plugins:** Leverage or develop OpenSearch plugins specifically for audit compliance features.

### 8.2 Log Collection Alternatives

The following audit log collection alternatives are included here for comparison purposes.

Fluent Bit is a lightweight, high-performance log forwarder optimized for resource-constrained environments and edge collection, while Fluentd is a more feature-rich, Ruby-based log processor designed for complex data transformations and centralized aggregation scenarios.

OpenTelemetry is a vendor-neutral, open-source observability framework that provides standardized APIs, SDKs, and tools for collecting, processing, and exporting telemetry data (logs, metrics, and traces) from applications and infrastructure.

Vector is a high-performance, Rust-based observability data pipeline that excels in audit log architectures by providing advanced data transformation, validation, and routing capabilities with strong delivery guarantees.

**Choose Fluent Bit when:**
- Need minimal resource overhead
- High throughput performance is critical
- Simple log collection and forwarding
- Edge or IoT environments
- Kubernetes sidecar pattern
- Basic compliance requirements
- Deploying on edge devices or containers

**Choose Vector when:**
- Complex data transformations required
- Strict compliance and data validation needs
- Multiple data sources and destinations
- Advanced routing and filtering logic
- Performance-critical environments

**Choose OpenTelemetry when:**
- Building comprehensive observability stack
- Need correlation between logs, metrics, and traces
- Distributed microservices architecture
- Vendor-neutral telemetry collection
- Long-term observability strategy

### 8.3 NATS vs Data Prepper

Since NATS is an existing component used in the Tazama stack the section below includes a comparison between NATS and Data Prepper.

NATS JetStream can be used instead of Data Prepper in an audit logging architecture with Fluent Bit and OpenSearch. NATS is listed as one of the official output plugins in Fluent Bit integration and is straightforward and officially supported.

NATS is more of an infrastructure component that enables flexible, scalable messaging architectures, while Data Prepper is a purpose-built tool for log processing and OpenSearch ingestion.

The choice depends on specific requirements:

- If you need a simple, direct path from Fluent Bit to OpenSearch with basic transformations, Data Prepper is likely the better choice
- If you're building a more complex system that needs message durability, multiple consumers, or event-driven processing, NATS provides more architectural flexibility

#### 8.3.1 Architecture Flow

```
Fluent Bit → NATS → Consumer Service → OpenSearch
Instead of: Fluent Bit → Data Prepper → OpenSearch
```

#### 8.3.2 Benefits of Using NATS

**Message Durability & Reliability**
- NATS Streaming/JetStream provides message persistence and delivery guarantees
- Better handling of backpressure and temporary downstream failures
- Message replay capabilities for recovery scenarios

**Decoupling & Scalability**
- Complete decoupling between log producers and consumers
- Multiple consumers can process the same log streams
- Horizontal scaling of processing components
- Load balancing across consumer instances

**Operational Flexibility**
- Can pause/resume processing without losing data
- Easy to add new consumers for different processing pipelines
- Better support for maintenance windows

**Implementation Approach**
1. Configure Fluent Bit to output to NATS subjects (topics)
2. Create consumer services that subscribe to NATS subjects and process logs
3. Consumer services handle transformation, enrichment, and forwarding to OpenSearch
4. Use NATS JetStream for persistence and delivery guarantees

#### 8.3.3 NATS Considerations

**Additional Complexity**
- Need to manage NATS infrastructure
- Requires building custom consumer services
- More moving parts to monitor

**Data Processing**
- Need to implement the data transformation logic that Data Prepper would have handled
- Consider using existing libraries or frameworks for log processing in consumer services

**NATS Configuration**
- Use JetStream for audit logs to ensure durability
- Configure appropriate retention policies
- Set up clustering for high availability

This architecture is particularly well-suited for high-volume audit logging scenarios where the following features are required:

- Guaranteed message delivery
- Ability to replay messages
- Fan out logs to multiple processing pipelines

#### 8.3.4 NATS vs Data Prepper Comparison

For most straightforward audit logging scenarios, Data Prepper's simplicity and purpose-built nature for log processing makes it the more practical choice, despite NATS being more powerful for complex streaming architectures.

| Aspect | NATS | Data Prepper |
|---|---|---|
| Primary Purpose | Message broker/streaming platform | Data transformation and ingestion pipeline |
| Architecture Role | Message queue/event streaming | ETL processor and router |
| Message Persistence | JetStream provides durable messaging | Temporary buffering during processing |
| Data Transformation | Requires custom consumer logic | Built-in processors (grok, mutate, drop, etc.) |
| OpenSearch Integration | Requires custom consumer with OpenSearch client | Native OpenSearch sink with bulk operations |
| Configuration | NATS server config + consumer application code | YAML pipeline configuration |
| Scalability | Horizontal scaling via clustering | Vertical scaling, limited horizontal options |
| Message Delivery | At-least-once, exactly-once (with JetStream) | At-least-once delivery guarantees |
| Backpressure Handling | Queue-based buffering with flow control | Built-in backpressure with retries |
| Monitoring | NATS monitoring + custom consumer metrics | Built-in metrics and OpenSearch integration |
| Development Effort | High (custom consumers required) | Low (configuration-driven) |
| Operational Complexity | Medium to High (NATS cluster + consumers) | Low to Medium (single component) |
| Memory Usage | Low (lightweight broker) | Higher (processing buffers and transformations) |
| Latency | Very low (sub-millisecond) | Higher due to processing overhead |
| Multi-consumer Support | Native (pub/sub, fan-out) | Limited (single pipeline output) |
| Message Replay | Yes (JetStream retention policies) | No (processed messages are forwarded) |
| Schema Evolution | Flexible (consumer handles changes) | Requires pipeline reconfiguration |
| Error Handling | Custom implementation in consumers | Built-in dead letter queues and retry logic |
| Community & Ecosystem | Growing, cloud-native focused | AWS-centric, OpenSearch ecosystem |
| License | Apache 2.0 (open source) | Apache 2.0 (open source) |
| Deployment Model | Separate infrastructure component | Embedded in logging pipeline |
| Resource Requirements | Separate compute/memory for NATS + consumers | Single process resource allocation |
| Use Case Fit | High-volume, multi-consumer, real-time streaming | Simple ETL, direct OpenSearch ingestion |
| Maintenance Overhead | Higher (multiple components) | Lower (single pipeline) |
| Debugging Complexity | Higher (distributed tracing needed) | Lower (centralized processing) |
| Cost Implications | Additional infrastructure + development time | Minimal additional infrastructure |

**Recommendation Summary**

Choose NATS when:
- Need multiple consumers for the same log data
- Require message replay/reprocessing capabilities
- High-volume streaming with strict latency requirements
- Need to decouple log producers from processors
- Planning complex event-driven architectures

Choose Data Prepper when:
- Simple, direct log transformation and OpenSearch ingestion
- Minimal operational complexity requirements
- Quick implementation timeline
- Standard log processing patterns
- Primarily single-destination logging pipeline

### 8.4 Audit Logging Architecture Comparison

The following section compares how the proposed architecture compares to the Grafana Observability Stack (LGTM)

**Stack A: Fluent Bit + Data Prepper + OpenSearch**
- Components: Fluent Bit (collection) → Data Prepper (processing) → OpenSearch (storage) → OpenSearch Dashboards (visualization)

**Stack B: Grafana Observability Stack (LGTM)**
- Components: Promtail/Grafana Agent (collection) → Loki (logs) + Tempo (traces) + Mimir (metrics) → Grafana (visualization)

#### 8.4.1 Advantages: Fluent Bit + Data Prepper + OpenSearch

**Audit-Specific Strengths**
- Full-text search excellence: OpenSearch provides superior full-text search capabilities essential for audit log investigations
- Structured document storage: Native JSON document handling perfect for structured audit events
- Compliance-focused: Data Prepper designed specifically for log processing with audit requirements in mind
- Mature audit tooling: Established ecosystem of security plugins and SIEM integrations
- Advanced querying: Elasticsearch Query DSL enables complex audit queries and correlations

**Operational Benefits**
- Resource efficiency: Fluent Bit's minimal footprint ideal for audit log collection at scale
- Proven stability: Well-established stack with extensive production deployments
- Security features: Built-in security plugins, field-level security, and audit trails
- Data retention: Sophisticated index lifecycle management for compliance requirements

#### 8.4.2 Disadvantages: Fluent Bit + Data Prepper + OpenSearch

**Limitations**
- Single-purpose focus: Primarily logs-only; limited metrics and tracing correlation
- Complex cluster management: Requires expertise in OpenSearch cluster operations
- Limited observability correlation: Difficult to correlate audit logs with application performance metrics
- Vendor considerations: OpenSearch licensing and support considerations

**Operational Challenges**
- Resource intensive storage: OpenSearch requires significant compute and storage resources
- Index management complexity: Requires careful index design and management for performance
- Limited real-time capabilities: Less optimized for real-time streaming analytics

#### 8.4.3 Advantages: Grafana Observability Stack

**Comprehensive Observability**
- Unified correlation: Native ability to correlate audit logs with traces and metrics in single interface
- Cost-effective logs: Loki's label-based approach significantly reduces storage costs
- Real-time streaming: Excellent real-time log streaming and alerting capabilities
- Modern UI/UX: Superior visualization and dashboard capabilities through Grafana
- Cloud-native design: Built for modern containerized and cloud environments

**Operational Excellence**
- Simplified operations: Less complex than managing OpenSearch clusters
- Better resource utilization: More efficient resource usage, especially for logs
- Vendor neutrality: Completely open source with no licensing concerns
- Active development: Rapid feature development and community support
- Multi-tenancy: Built-in multi-tenant capabilities for organizational separation

#### 8.4.4 Disadvantages: Grafana Observability Stack

**Audit-Specific Limitations**
- Limited full-text search: Loki's label-based approach not ideal for complex audit log searches
- Query complexity: LogQL learning curve and limitations compared to Elasticsearch DSL
- Compliance gaps: Less mature ecosystem for specific compliance and regulatory requirements
- SIEM integration: Fewer native integrations with enterprise SIEM platforms
- Audit trail features: Less sophisticated audit trail and security features

**Operational Concerns**
- Multiple components: Managing Loki, Tempo, Mimir, and Grafana increases operational complexity
- Data retention challenges: More complex data retention policies across multiple systems
- Limited enterprise features: Fewer enterprise security and compliance features out-of-the-box
- Search performance: Poor performance for ad-hoc searches across large time ranges

#### 8.4.5 Use Case Recommendations

**Choose Fluent Bit + Data Prepper + OpenSearch for:**
- Compliance-heavy environments (SOX, PCI-DSS, HIPAA)
- Security-focused organizations requiring advanced audit capabilities
- Large-scale log search requirements
- Regulatory reporting and forensic investigations
- Existing Elasticsearch/OpenSearch expertise
- SIEM integration requirements

**Choose Grafana Observability Stack for:**
- Modern cloud-native applications
- Cost-sensitive environments with high log volumes
- Developer-focused organizations prioritizing observability correlation
- Real-time monitoring and alerting requirements
- Open source preference with no vendor lock-in concerns
- Microservices architectures requiring distributed tracing correlation

**Hybrid Approach Considerations**

Many organizations adopt hybrid strategies:
- Primary audit logs: OpenSearch for compliance and investigation
- Operational observability: Grafana stack for application monitoring
- Data routing: Use both stacks with different log streams based on requirements
- Graduated approach: Start with Grafana stack, add OpenSearch for specific audit requirements

**Summary**

- OpenSearch Stack: Better for traditional audit logging with strong search, compliance, and security investigation requirements.
- Grafana Stack: Better for modern observability-driven organizations that need cost-effective logging with application performance correlation.
- The choice depends primarily on whether audit logging needs emphasize compliance and investigation (OpenSearch) or operational observability and cost efficiency (Grafana stack).

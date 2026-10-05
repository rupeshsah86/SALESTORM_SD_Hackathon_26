# 02.4 High-Level Design — Deployment Diagram (Kubernetes Infrastructure Topology)

## Overview
The Deployment Diagram defines the physical and cloud infrastructure topology for **SALESTORM 2026**. Deployed on Managed Kubernetes (AWS EKS / GCP GKE) across three Availability Zones (AZ-a, AZ-b, AZ-c), the architecture isolates ingress traffic, stateless microservices, and stateful datastores to maintain zero single-points-of-failure (SPOF) and sub-second scaling.

---

## Kubernetes Infrastructure Deployment Diagram

```mermaid
graph TB
    subgraph Internet ["🌐 Public Internet"]
        Traffic Surge["10,000 Concurrent Buyers<br/>(HTTP/2 & HTTPS Traffic)"]
    end

    subgraph EdgeCloud ["☁️ Cloudflare Edge Infrastructure"]
        EdgeWAF["Cloudflare Anycast WAF / Edge CDN<br/>(DDoS Mitigation & Virtual Waiting Room)"]
    end

    subgraph CloudVPC ["🛡️ Cloud VPC (10.0.0.0/16)"]
        
        subgraph PublicSubnets ["Public Subnets (AZ-a, AZ-b, AZ-c)"]
            ALB["AWS Application Load Balancer / NLB<br/>(Multi-AZ Layer 4/7 Load Balancing)"]
        end

        subgraph K8sCluster ["☸️ Kubernetes Cluster (EKS / GKE)"]
            
            subgraph GatewayNodePool ["Node Pool 1: Edge & Ingress (c6i.2xlarge - Multi-AZ)"]
                KongPod1["Kong API Gateway Pod (AZ-a)"]
                KongPod2["Kong API Gateway Pod (AZ-b)"]
                KongPod3["Kong API Gateway Pod (AZ-c)"]
            end

            subgraph AppNodePool ["Node Pool 2: Microservices (c6i.4xlarge - Autoscale 5 -> 30 Nodes)"]
                InvPod1["Inventory Svc Pod 1 (AZ-a)"]
                InvPod2["Inventory Svc Pod 2 (AZ-b)"]
                
                OrderPod1["Order Svc Pod 1 (AZ-a)"]
                OrderPod2["Order Svc Pod 2 (AZ-c)"]
                
                PaymentPod1["Payment Svc Pod 1 (AZ-b)"]
                PaymentPod2["Payment Svc Pod 2 (AZ-c)"]
            end
            
            HPA["Horizontal Pod Autoscaler (HPA)<br/>CPU > 60% OR RPS > 2000 -> Scale Pods"]
        end

        subgraph PrivateDataSubnets ["Private Data Subnets (AZ-a, AZ-b, AZ-c)"]
            subgraph ElastiCache ["⚡ AWS ElastiCache for Redis (Cluster Mode Enabled)"]
                RedisPrimary1["Redis Master Shard 1 (AZ-a)"]
                RedisPrimary2["Redis Master Shard 2 (AZ-b)"]
                RedisReplica1["Redis Replica 1 (AZ-c)"]
            end

            subgraph ManagedPostgres ["🐘 AWS Aurora PostgreSQL Serverless v2 (Multi-AZ)"]
                DBPrimary["Aurora Primary Writer (AZ-a)"]
                DBReader1["Aurora Read Replica 1 (AZ-b)"]
                DBReader2["Aurora Read Replica 2 (AZ-c)"]
            end

            subgraph ManagedKafka ["📡 AWS MSK Kafka Cluster (3 Broker Nodes)"]
                KafkaBroker1["Kafka Broker 1 (AZ-a)"]
                KafkaBroker2["Kafka Broker 2 (AZ-b)"]
                KafkaBroker3["Kafka Broker 3 (AZ-c)"]
            end
        end

    end

    %% Wiring Connections
    Traffic Surge --> EdgeWAF
    EdgeWAF --> ALB
    ALB --> KongPod1 & KongPod2 & KongPod3
    
    KongPod1 & KongPod2 & KongPod3 --> InvPod1 & InvPod2
    KongPod1 & KongPod2 & KongPod3 --> OrderPod1 & OrderPod2
    KongPod1 & KongPod2 & KongPod3 --> PaymentPod1 & PaymentPod2

    InvPod1 & InvPod2 --> RedisPrimary1 & RedisPrimary2
    InvPod1 & InvPod2 --> DBPrimary

    OrderPod1 & OrderPod2 --> DBPrimary
    OrderPod1 & OrderPod2 --> DBReader1
    OrderPod1 & OrderPod2 --> KafkaBroker1

    PaymentPod1 & PaymentPod2 --> DBPrimary
    PaymentPod1 & PaymentPod2 --> KafkaBroker2

    %% Styling
    classDef cloudStyle fill:#0f172a,stroke:#334155,stroke-width:2px,color:#fff;
    classDef k8sStyle fill:#1e3a8a,stroke:#1d4ed8,stroke-width:2px,color:#fff;
    classDef dbStyle fill:#78350f,stroke:#b45309,stroke-width:2px,color:#fff;

    class Traffic Surge,EdgeWAF,ALB cloudStyle;
    class KongPod1,KongPod2,KongPod3,InvPod1,InvPod2,OrderPod1,OrderPod2,PaymentPod1,PaymentPod2,HPA k8sStyle;
    class RedisPrimary1,RedisPrimary2,RedisReplica1,DBPrimary,DBReader1,DBReader2,KafkaBroker1,KafkaBroker2,KafkaBroker3 dbStyle;
```

---

## Infrastructure Topology & High-Availability Configurations

### 1. Multi-AZ High Availability Design
- All Node Pools and managed datastores are strictly distributed across 3 distinct Availability Zones (`us-east-1a`, `us-east-1b`, `us-east-1c`).
- **Pod Anti-Affinity Rules**: Prevents pods of the same microservice (e.g., `Inventory Service`) from running on the same physical Kubernetes worker node or single AZ.

### 2. Auto-scaling & Resource Allocation

| Component | Minimum Pods | Peak Burst Pods | Scaling Metric Trigger | CPU / Memory Request |
| :--- | :--- | :--- | :--- | :--- |
| **Kong API Gateway** | 6 Pods | 20 Pods | HTTP RPS $> 1,500$ per pod | 2 vCPU / 4 GB RAM |
| **Flash Inventory Service** | 10 Pods | 40 Pods | CPU Usage $> 60\%$ or latency $> 50\text{ms}$ | 4 vCPU / 8 GB RAM |
| **Checkout & Order Service**| 4 Pods | 25 Pods | CPU Usage $> 70\%$ | 4 vCPU / 8 GB RAM |
| **Payment Service** | 4 Pods | 20 Pods | Request Queue Depth $> 100$ | 2 vCPU / 4 GB RAM |

### 3. Managed Datastore Topologies
- **AWS ElastiCache for Redis**: 3 Shards with 1 Primary + 1 Read Replica each. In-Memory Engine v7.0 with AOF enabled for durability.
- **AWS Aurora PostgreSQL**: Multi-AZ Writer instance with 2 Read Replicas. Auto-scaling storage up to 64TB with Read Replica auto-failover ($< 30\text{s}$ RTO).
- **AWS MSK (Kafka)**: 3 Brokers, Partition factor of 6 per topic, Replication Factor of 3 (`min.insync.replicas = 2`).

---
*Document Version: 1.0.0 — SALESTORM 2026 Deployment Architecture*

# GCP Infrastructure Requirements Documentation

## Table of Contents

1. [Overview](#overview)
2. [Business Objectives](#business-objectives)
3. [System Architecture](#system-architecture)
4. [Infrastructure Components](#infrastructure-components)
5. [Client Onboarding Process](#client-onboarding-process)
6. [Resource Requirements](#resource-requirements)
7. [Dependencies and Integration Points](#dependencies-and-integration-points)
8. [Security Requirements](#security-requirements)
9. [Operational Requirements](#operational-requirements)
10. [Non-Functional Requirements](#non-functional-requirements)
11. [Constraints and Assumptions](#constraints-and-assumptions)

---

## Overview

This repository automates the creation and management of Google Cloud Platform (GCP) infrastructure for client onboarding. Each new client receives a dedicated GCP project with all necessary resources to deploy and operate the platform on their custom domain.

### Purpose

The infrastructure automation system enables:
- **Automated Project Creation**: Provision a complete GCP project for each new client
- **Infrastructure as Code**: All resources defined and managed via Terraform
- **Consistent Deployments**: Standardized infrastructure across all client projects
- **CI/CD Integration**: Automated infrastructure provisioning via GitHub Actions
- **Multi-Environment Support**: Separate Dev and Prod environments per client


## Business Objectives

### Primary Goals

1. **Automate Client Onboarding**
   - Ensure consistent infrastructure across all clients

2. **Enable Multi-Tenancy**
   - Isolate each client in their own GCP project
   - Provide dedicated resources per client
   - Support custom domain configuration per client

3. **Support Platform Deployment**
   - Create infrastructure ready for containerized application deployment
   - Provide Artifact Registry for container image storage
   - Enable Kubernetes-based workload deployment

4. **Ensure Scalability**
   - Support unlimited number of client projects
   - Enable independent scaling per client
   - Maintain performance isolation between clients

### Success Criteria

- New client project can be provisioned
- Zero manual infrastructure configuration steps
- 100% infrastructure coverage via Terraform
- All clients have identical base infrastructure capabilities

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Onboarding Flow                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              GitHub Actions Workflow                         │
│  • Terraform Plan/Apply                                      │
│  • Resource Validation                                       │
│  • State Management                                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Terraform Infrastructure Code                   │
│  • Project Creation                                         │
│  • Resource Provisioning                                    │
│  • Configuration Management                                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              GCP Project (Per Client)                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Networking (VPC, Subnets, Firewall Rules)          │   │
│  │  GKE Cluster (Autopilot)                            │   │
│  │  Artifact Registry                                   │   │
│  │  Service Accounts & IAM                              │   │
│  │  Cloud Storage (GCS)                                 │   │
│  │  BigQuery Datasets                                   │   │
│  │  Pub/Sub Topics & Subscriptions                     │   │
│  │  Secret Manager                                      │   │
│  │  Static IPs & Load Balancing                         │   │
│  │  Cloud Armor & SSL Policies                          │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Platform Repository                             │
│  • Builds container images                                  │
│  • Pushes to Artifact Registry                              │
│  • Deploys to GKE Cluster                                   │
└─────────────────────────────────────────────────────────────┘
```


## Infrastructure Components

### 1. GCP Project

**Purpose**: Isolated GCP project for each client

**Requirements**:
- Unique project ID following naming convention: `{ProjectNumber}-{ProjectName}`
- Project number format: `D{Number}` or `M{Number}` (e.g., D112558, M15646)
- Billing account association
- Organization/folder structure support
- Resource labels for tracking and organization

**Configuration**:
- Project ID: Configurable via Terraform variables
- Billing Account: (We have our internal cloud team helping with this)
- Folder ID: Platform or Non-Platform

### 2. Networking Infrastructure

**Purpose**: Secure network connectivity for GKE cluster and workloads

**Components**:

#### 2.1 VPC Network
- **Name**: Configurable (default: `{project-name}-{env}-vpc`)
- **Routing Mode**: Global
- **Purpose**: Isolated network for client resources

#### 2.2 Subnets
- **Primary Subnet**: 
  - CIDR: Configurable (default: `10.10.0.0/20`)
  - Region: Configurable (default: `us-west2`)
  - Private Google Access: Enabled
  - Flow Logs: Enabled
- **Secondary IP Ranges** (for GKE):
  - Pods CIDR: Configurable (default: `10.20.0.0/14`)
  - Services CIDR: Configurable (default: `10.24.0.0/20`)

**Requirements**:
- Non-overlapping CIDR blocks across all client projects
- Support for regional and multi-region deployments
- Private Google Access for GKE nodes
- VPC Flow Logs for security monitoring

### 3. Google Kubernetes Engine (GKE)

**Purpose**: Container orchestration platform for application workloads

**Cluster Configuration**:
- **Type**: Autopilot (fully managed)
- **Location**: Regional (multi-zone) - Must be us-west2
- **Release Channel**: STABLE
- **Datapath Provider**: ADVANCED_DATAPATH
- **Networking Mode**: VPC-native

**Features Enabled**:
- Workload Identity
- Vertical Pod Autoscaling (VPA)
- Horizontal Pod Autoscaling (HPA)
- Cloud DNS integration
- Gateway API (Standard channel)
- Managed Prometheus
- Advanced Datapath Observability
- GCE Persistent Disk CSI Driver
- GCS Fuse CSI Driver

### 4. Artifact Registry

**Purpose**: Container image storage for platform deployments

**Configuration**:
- **Repository Name**: Configurable (default: `collector-images`)
- **Format**: DOCKER
- **Location**: Same region as GKE cluster
- **Description**: Docker repository for client container images

**Requirements**:
- Repository must exist before platform repository pushes images
- Integration with GKE for image pulling
- IAM permissions for platform repository CI/CD

### 5. Service Accounts and IAM

**Purpose**: Secure access control for workloads and services

#### 5.1 Frontend Service Account
- **Name**: `frontend-sa`
- **Roles**:
  - `roles/secretmanager.secretAccessor`
  - `roles/storage.objectViewer`
- **Workload Identity**: Enabled (binds to Kubernetes service account)

#### 5.2 Backend Service Accounts

**Workflow API Service Account** (`workflow-api-sa`):
- Roles: `pubsub.publisher`, `storage.objectCreator`, `logging.logWriter`, `secretmanager.secretAccessor`

**Sitemap Analyzer Service Account** (`sitemap-analyzer-sa`):
- Roles: `pubsub.publisher`, `storage.objectCreator`, `logging.logWriter`, `secretmanager.secretAccessor`

**Analytics Service Account** (`analytics-sa`):
- Roles: `pubsub.subscriber`, `pubsub.publisher`, `bigquery.dataEditor`, `bigquery.jobUser`, `storage.objectViewer`, `logging.logWriter`, `secretmanager.secretAccessor`

**Collector Service Account** (`collector-sa`):
- Roles: `pubsub.publisher`, `pubsub.subscriber`, `storage.objectCreator`, `logging.logWriter`, `secretmanager.secretAccessor`

**Collector Loader Service Account** (`collector-loader-sa`):
- Roles: `pubsub.subscriber`, `bigquery.dataEditor`, `bigquery.jobUser`, `storage.objectViewer`, `logging.logWriter`, `secretmanager.secretAccessor`

### 6. Cloud Storage (GCS)

**Purpose**: Object storage for raw data and entities

#### 6.1 Raw Data Bucket
- **Naming**: `cra-raw-data-prod-{project-name}` (configurable)
- **Location**: Same region as cluster
- **Features**:
  - Versioning: Enabled
  - Uniform bucket-level access: Enabled
  - Lifecycle rule: Delete after 365 days
  - Force destroy: Disabled (production safety)

#### 6.2 Entities Bucket
- **Naming**: `entities-{project-name}` (configurable)
- **Location**: Same region as cluster
- **Features**:
  - Versioning: Enabled
  - Uniform bucket-level access: Enabled

**IAM Bindings**:
- Analytics service account: Object Viewer on both buckets
- Collector service accounts: Object Creator on raw data bucket

### 7. BigQuery

**Purpose**: Data warehouse for analytics and reporting

**Configuration**:
- **Dataset ID**: Configurable (default: `cra_data` or `{project_number}_{project_name}`)
- **Location**: Same region as cluster (default)
- **Default Table Expiration**: 1 year
- **Default Partition Expiration**: 1 year
- **Access Control**: Project owners have OWNER role

### 8. Pub/Sub

**Purpose**: Event-driven messaging for microservices communication

**Subscription Configuration**:
- Ack deadline: 60-600 seconds (varies by subscription)
- Message retention: 10 minutes
- Retry policy: Exponential backoff (10s min, 600s max)
- Message ordering: Enabled where applicable

### 9. Secret Manager

**Purpose**: Secure storage for sensitive configuration


### 10. Static IP and Load Balancing

**Purpose**: External access to frontend application

#### 10.1 Global Static IP
- **Name**: `frontend-static-ip` (configurable)
- **Type**: EXTERNAL
- **IP Version**: IPv4
- **Purpose**: Reserved IP for frontend ingress

### 11. Cloud Armor Security Policy

**Purpose**: DDoS protection and rate limiting

**Configuration**:
- **Name**: `frontend-security-policy` (configurable)
- **Default Rule**: Allow all traffic
- **Rate Limiting Rule**:
  - Threshold: Configurable (default: 100 requests)
  - Interval: Configurable (default: 60 seconds)
  - Action: Throttle (conform: allow, exceed: deny 429)
  - Enforcement: Per IP address

### 12. SSL Policy

**Purpose**: TLS/SSL configuration for secure ingress

**Configuration**:
- **Name**: `modern-ssl-policy` (configurable)
- **Profile**: MODERN
- **Minimum TLS Version**: TLS 1.2 (configurable)
- **Purpose**: Applied to HTTPS load balancer


### 13. Kubernetes Resources

**Purpose**: Application deployment configuration

#### 13.1 Namespace
- **Name**: Configurable (default: `streaming`)
- **Labels**: Environment, managed-by, team
- **Purpose**: Isolate client workloads

#### 13.2 ConfigMaps
- **Frontend ConfigMap**: Application configuration
  - NextAuth URL
  - Auth0 issuer
  - Service URLs (workflow-api, sitemap-analyzer, manual-browser)
  - GCP project and BigQuery dataset references
- **Pub/Sub ConfigMap**: Pub/Sub topic and subscription names
- **Service-specific ConfigMaps**: Per-service configuration

#### 13.3 Kubernetes Secrets
- **Frontend Secrets**: Auth0 and NextAuth credentials
- **Service Account Secrets**: Workload Identity bindings

#### 13.4 Deployments and Services
- Defined in `kubernetes.tf` (detailed configuration)
- All services configured with:
  - Resource limits and requests
  - Health checks
  - Environment variables from ConfigMaps/Secrets
  - Service account bindings for Workload Identity

**Requirements**:
- All Kubernetes resources created via Terraform
- Support for rolling updates
- Health check endpoints configured
- Resource quotas per namespace

### 14. GCP APIs

**Required APIs** (automatically enabled):
- `compute.googleapis.com`: Compute Engine
- `container.googleapis.com`: Kubernetes Engine
- `artifactregistry.googleapis.com`: Artifact Registry
- `iam.googleapis.com`: Identity and Access Management
- `secretmanager.googleapis.com`: Secret Manager
- `storage-api.googleapis.com`: Cloud Storage
- `cloudresourcemanager.googleapis.com`: Resource Manager
- `servicenetworking.googleapis.com`: Service Networking
- `pubsub.googleapis.com`: Pub/Sub
- `bigquery.googleapis.com`: BigQuery
- `logging.googleapis.com`: Cloud Logging
- `monitoring.googleapis.com`: Cloud Monitoring

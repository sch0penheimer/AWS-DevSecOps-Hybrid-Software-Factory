
---
> **Last Updated:** January 6th, 2025
>
> **Author:** [Haitam Bidiouane](https://linkedin.com/in/haitam-bidiouane/)
---

# AWS DevSecOps Hybrid Software Factory

This project implements a fully automated Hybrid DevSecOps Factory/Platform on AWS, designed to enforce security and compliance at every stage of the software delivery lifecycle, while decoupling <ins>**the CI/CD pipeline and related resources**</ins> from <ins>**the main platform**</ins> that hosts ECS EC2-based containerized application workloads, respectively via AWS CloudFormation and Terraform, hence the "Hybrid" label.

> [!NOTE]
> Architected, implemented, and fully documented by **[Haitam Bidiouane](https://linkedin.com/in/haitam-bidiouane/)** (***@sch0penheimer***).

<div align="center">

![](./doc/Metadoc/GitHub-Intro-Banner.png)

</div>

## Table of Contents

### [Section I: Factory Architecture & Infrastructure Overview](#section-1-factory-architecture--infrastructure-overview)
- [Project Overview](#project-overview)
- [Architecture](#architecture)
  - [Hybrid IaC Approach](#hybrid-iac-approach)
  - [High-level AWS Architecture](#high-level-aws-architecture)

### [Section II: Architectural Deep Dive](#section-2-architectural-deep-dive)
- [Terraform Infrastructure Sub-Architecture](#terraform-infrastructure-sub-architecture)
  - [VPC Architecture](#vpc-architecture)
  - [Public Resources Architecture](#public-resources-architecture)
  - [Private Resources Architecture](#private-resources-architecture)
- [AWS CloudFormation CI/CD Sub-Architecture](#aws-cloudformation-cicd-sub-architecture)
  - [CodePipeline Architecture](#codepipeline-architecture)
  - [Pipeline Integration & Complete CI/CD Workflow](#pipeline-integration--complete-cicd-workflow)

### [Section III: Technical Implementation Details & Operations](#section-3-technical-implementation-details--operations)
- [Modular Terraform Approach](#modular-terraform-approach)
  - [Global Variables & Outputs](#global-variables--outputs)
  - [Network Module](#network-module)
    - [Security Groups & Firewalling Strategy](#security-groups--firewalling-strategy)
    - [Custom NAT EC2 instances](#custom-nat-ec2-instances)
  - [Compute Module](#compute-module)
    - [ECS Staging & Production Clusters](#ecs-staging--production-clusters)
    - [ECS EC2 Launch Template](#ecs-ec2-launch-template)
    - [Task Definitions](#task-definitions)
  - [Storage Module](#storage-module)
    - [S3 Artifact Store](#s3-artifact-store)
    - [S3 Lambda Packaging Bucket](#s3-lambda-packaging-bucket)

- [AWS CloudFormation template](#aws-cloudformation-template)
  - [CI/CD Workflow](#cicd-workflow)
    - [AWS CodeConnections Connection](#aws-codeconnections-connection)
    - [Normalization & Aggregation Lambda Function](#normalization--aggregation-lambda-function)
    - [AWS CodeBuild Projects](#aws-codebuild-projects)
    - [Blue/Green Deployment Strategy](#bluegreen-deployment-strategy)
  - [Security & Compliance](#security--compliance)
    - [SSM Parameter Store](#ssm-parameter-store)
    - [Encryption & KMS](#encryption--kms)
    - [Secrets Scanning (git-secrets)](#secrets-scanning-git-secrets)
    - [SAST - Static Application Security Analysis (Snyk)](#sast---static-application-security-analysis-snyk)
    - [SCA - Software Composition Analysis (Clair)](#sca---software-composition-analysis-clair)
    - [DAST - Dynamic Application Security Analysis (OWASP ZAP)](#dast---dynamic-application-security-analysis-owasp-zap)
    - [RASP - Runtime Application Security Protection (CNCF Falco)](#rasp---runtime-application-security-protection-cncf-falco)
  - [Event-Driven Architecture](#event-driven-architecture)
    - [AWS EventBridge Rules](#aws-eventbridge-rules)
    - [AWS CloudWatch Events](#aws-cloudwatch-events)
    - [SNS Topics & Subscriptions](#sns-topics--subscriptions)  
  - [Monitoring & Observability](#monitoring--observability)
    - [AWS CloudWatch Dedicated Log Groups](#aws-cloudwatch-dedicated-log-groups)
    - [AWS CloudTrail & AWS Config](#aws-cloudtrail--aws-config)

### [Section IV: Deployment & Configuration Guide](#section-4-deployment--configuration-guide)
- [Deployment Scripts](#deployment-scripts)
- [Environment Configuration Reference](#environment-configuration-reference)

---

- [Fully Documented Deployment Walkthrough](#fully-documented-deployment-walkthrough)

<br/>

# Section 1: Factory Architecture & Infrastructure Overview

## Project Overview

This AWS DevSecOps Hybrid CI/CD Factory represents a <ins>**DevSecOps Software Factory**</ins>, an evolved approach to software delivery that extends traditional DevOps practices by embedding security controls throughout the entire software development lifecycle. The factory concept provides a standardized, automated environment for building, testing, and deploying software with security as a first-class citizen rather than an afterthought.

- **Development**: Secure coding practices integrated from initial commit with automated pre-commit hooks and static analysis
- **Security**: Continuous security scanning through SAST, SCA, DAST, and RASP tools embedded in pipeline stages & production environment
- **Operations**: Infrastructure security hardening and runtime monitoring with automated incident response

---

The platform also introduces a novel <ins>**hybrid IaC approach**</ins> that strategically separates infrastructure concerns based on resource characteristics and lifecycle management requirements. This separation provides optimal tooling selection for different infrastructure layers.

Also, the platform is specifically architected for ***AWS Free Tier compatibility***, enabling immediate deployment without incurring charges for evaluation and small-scale production workloads.

---

## Architecture
### Hybrid IaC Approach

This platform implements a *strategic separation of Infrastructure as Code responsibilities* between <ins>**Terraform**</ins> and <ins>**AWS CloudFormation**</ins>, creating a hybrid model that leverages the strengths of each tool while maintaining clear boundaries of concern.

<div align="center">

![Hybrid IaC Architecture](./doc/Metadoc/hybrid_iac.png)

*Figure 1: Hybrid Infrastructure as Code Architecture - Separation of concerns and integration between Terraform and AWS CloudFormation*

</div>

**I) Terraform Infrastructure Layer:**
Manages foundational, reusable infrastructure components and provisions core network and compute resources. This layer is optimized for long‑lived infrastructure that requires complex dependency handling and reliable state tracking

**II) CloudFormation Pipeline Layer:**
Manages orchestration of AWS-native services and pipeline-specific resources, this CloudFormation layer leverages native AWS integrations and built‑in drift detection to ensure reliable, maintainable pipeline lifecycle management.

### Cross-IaC Integration Pattern

1. **Terraform Deployment**: Deployment scripts execute `terraform plan` and `terraform apply` for infrastructure provisioning
2. **Output Capture**: Scripts capture Terraform outputs (VPC ID, subnet IDs, security group IDs) programmatically
3. **CloudFormation Orchestration**: Scripts initiate CloudFormation stack creation with captured Terraform outputs as parameter inputs
4. **Runtime Integration**: CloudFormation stack receives infrastructure identifiers and provisions pipeline resources with proper resource references

**Integration Architecture:**
```
          Deployment Script → Terraform Apply → Capture Outputs → CloudFormation Deploy
                  ↓                ↓                   ↓                   ↓
            Script Logic     Infrastructure    VPC ID, Subnets    Pipeline Resources
            Orchestration    Provisioning      Security Groups,    with References
                                                ECS Clusters, 
                                              Task definitions ...
```

- **Phase 1**: The provided Deployment Scripts execute Terraform deployment and wait for completion
- **Phase 2**: Scripts parse Terraform state or output files to extract resource identifiers
- **Phase 3**: Scripts construct CloudFormation parameter mappings from Terraform outputs
- **Phase 4**: Scripts deploy CloudFormation stack with parameter values for seamless integration

---
### High-level AWS Architecture

<div align="center">

![AWS Platform Architecture](./assets/Main_Architecture/AWS_DevSecOps_Hybrid_CICD_Platform_Architecture.png)

*Figure 2: High-level AWS DevSecOps Factroy Architecture - Complete Software Factory Overview*

***(Click on the architecture for a better full-screen view)***

</div>

The AWS DevSecOps Hybrid CI/CD Factory implements a <ins>**multi-tier cloud-native architecture**</ins> that orchestrates secure software delivery through strategically decoupled infrastructure and pipeline layers. The platform leverages native AWS services to create a comprehensive DevSecOps environment with integrated security controls, automated compliance validation, and event-driven operational patterns.

**1. `Network Foundation Layer`:**
The platform implements a **Custom VPC Architecture** that isolates the network with thoughtfully segmented subnets across multiple Availability Zones to ensure high availability and fault tolerance, a **Custom NAT Strategy** that employs personalized EC2‑based NAT instances to provide controlled and <ins>FREE</ins> egress routing for resources in private subnets, and a **Load Balancing Infrastructure** using Application Load Balancers to efficiently distribute traffic across containerized application tiers.

**2. `Compute & Container Orchestration:`**
The platform uses distinct **ECS Cluster Architecture** by maintaining separate staging and production clusters running on EC2 instances managed by Auto Scaling Groups for dynamic capacity, and also leverages a secure **Container Registry** (Amazon ECR) for scalable image storage with built‑in vulnerability scanning, and implements a **Blue/Green Deployment** strategy to enable zero‑downtime releases by coordinating ECS service updates with ALB target‑group switching.

**3. `CI/CD Pipeline Infrastructure:`**
The factory relies on **AWS CodePipeline** as a multi‑stage orchestrator to integrate source control, coordinate builds, perform security scans, and automate deployments; **AWS CodeBuild** supplies isolated, containerized build environments for compiling code, executing security and compliance analyses, and producing artifacts; and centralized **S3 Artifact Management** handles artifact storage with versioning and lifecycle policies to manage pipeline dependencies.

**4. `Security Integration Layer:`**
**Multi-Stage Security Scanning** is implemented throughout the CI process: SAST, SCA, DAST, and RASP tools are embedded across pipeline stages & the prod environment to continuously detect code, dependency, runtime, and application-layer risks; **AWS Security Hub** centralizes findings while a custom Lambda function normalizes and correlates alerts for unified visibility and automated response;

**5. `Event-Driven Operations:`**
The platform uses **AWS EventBridge** for flexible event routing and automated incident‑response workflows, **CloudWatch Integration** for centralized monitoring, logging, and alerting across all components, and **SNS Topics** to distribute operational and security alerts across multiple channels.

**6. `Compliance & Governance:`**
**AWS Config** provides continuous configuration compliance checks and drift detection across resources, **CloudTrail Auditing** captures comprehensive API activity logs for security and compliance analysis, and **IAM Access Control** enforces least‑privilege access through role‑based permissions and explicit cross‑service trust relationships.

The architecture implements <ins>**separation of concerns**</ins> through distinct operational domains while maintaining seamless integration through AWS-native service communication patterns. Each architectural layer is designed for independent scaling, maintenance, and security policy enforcement.

<ins>**Detailed Technical Specifications:**</ins>

The next chapter provides comprehensive technical deep-dives into each architectural component, implementation details, and operational procedures. Each sub-architecture is documented with specific configuration parameters, security controls, and integration patterns

---

# Section 2: Architectural Deep Dive
## Terraform Infrastructure Sub-Architecture
<div align="center">

![](./assets/Pseudo_Architectures/TF-INFRA.png)

*Figure 3: Terraform Infrastructure Sub-Architecture - Complete foundational infrastructure layer managed by Terraform*

***(Click on the architecture for a better full-screen view)***

</div>

The Terraform Infrastructure Sub-Architecture establishes the foundational layer of the DevSecOps Factory, managing long-lived infrastructure components through declarative configuration. This layer implements a <ins>**network-centric design**</ins> that provides secure, scalable infrastructure primitives for containerized workloads while maintaining strict separation between public and private resources.

**`Infrastructure State Management:`**
- **Terraform State Backend**: Remote state storage with locking mechanisms for concurrent execution prevention
- **Output Variables**: Structured data export for CloudFormation integration including VPC IDs, subnet identifiers, and security group references
- **Resource Tagging**: Comprehensive tagging strategy for cost allocation, environment identification, and compliance tracking

**| <ins>Network Foundation</ins>:**
- **Custom VPC**: Isolated network environment with `/16` CIDR block providing 65,531 (65,536 - 5 IPs reserved by AWS) available IP addresses across multiple Availability Zones
- **Subnet Segmentation**: Strategic separation between public subnets for internet-facing resources and private subnets for application workloads
- **Route Table Management**: Dedicated routing configurations for public internet access and private subnet egress through custom NAT instances

**| <ins>Compute Infrastructure</ins>:**
- **ECS Cluster Provisioning**: Separate staging and production clusters with EC2 launch configurations optimized for containerized workloads
- **Auto Scaling Groups**: Dynamic capacity management with configurable scaling policies based on CPU utilization and memory consumption
- **Security Group Configuration**: Network-level access control with least-privilege rules for inter-service communication

**| <ins>Load Balancing & Traffic Management</ins>:**
- **Application Load Balancer**: Layer 7 load balancing with SSL termination and target group management for blue/green deployments
- **Target Group Configuration**: Health check definitions and traffic routing rules for ECS service integration
- **Custom NAT Instance Strategy**: Cost-optimized egress solution using EC2 instances instead of managed NAT gateways

<br/>

Next, I will dive deeper into each section ***(VPC, Public & Private scopes)**


---
### VPC Architecture
<div align="center">

![](./assets/Pseudo_Architectures/TF-INFRA_VPC.png)

*Figure 4: VPC Network Architecture - Multi-AZ VPC design with strategic subnet segmentation and routing configuration*

***(Click on the architecture for a better full-screen view)***

</div>

The VPC Architecture implements a <ins>**multi-tier network design**</ins> that provides secure isolation, high availability, and controlled traffic flow for the DevSecOps Factory. The network foundation establishes clear boundaries between public-facing and private resources while enabling secure communication patterns across availability zones.

**`Network Topology & CIDR Design:`**
- **VPC CIDR Block**: `/16` network (`10.0.0.0/16`) providing 65,531 (65,536 - 5 IPs reserved by AWS) usable IP addresses for comprehensive resource allocation
- **Multi-AZ Distribution**: Resources distributed across `us-east-1a` and `us-east-1b` availability zones for fault tolerance and high availability
- **Subnet Allocation Strategy**: CIDR space partitioned to support both current deployment requirements and future scaling needs

**`Subnet Segmentation Strategy:`**
- **Public Subnets**: Internet-accessible subnets (`10.0.1.0/24`, `10.0.2.0/24`) hosting load balancers and NAT instances.
- **Private Subnets**: Isolated subnets (`10.0.3.0/24`, `10.0.4.0/24`) containing ECS clusters, application workloads, and sensitive resources
- **Reserved Address Space**: Additional CIDR ranges reserved for database subnets, cache layers, and future service expansion

**`Routing Architecture:`**
- **Internet Gateway**: Single IGW providing **EGRESS** internet connectivity for public subnet resources
- **Public Route Tables**: Direct routing to Internet Gateway for public subnet traffic with explicit 0.0.0.0/0 routes
- **Private Route Tables**: Custom routing through NAT instances for private subnet egress while maintaining inbound isolation
- **Local Route Propagation**: Automatic VPC-local routing for inter-subnet communication within the `10.0.0.0/16` space

**`Security & Access Control:`**
- **Security Groups**: Instance-level stateful firewalls rules are established to EVERYTHING enabling precise traffic control between application tiers and **LEAST PRIVILEGE ACCESS**

**`DNS & Name Resolution:`**
Only the Application Load Balancer (ALB) exposes a ***<ins>regional DNS name</ins>*** for *inbound internet access*. The ALB is the single public entry point; backend ECS/EC2 instances and other resources remain in private subnets without public IPs and receive traffic only from the ALB.

- The ALB publishes a regional DNS name (for example: my-alb-xxxxx.us-east-1.elb.amazonaws.com). **DO NOT RELY ON ALB IP ADDRESSES, as they are dynamic and subject to change**.
- Enforce security by allowing inbound HTTP/HTTPS only on the ALB security group; backend security groups accept traffic only from the ALB SG. Private resources use NAT instances for outbound internet access, not public-facing IPs.

This pattern centralizes inbound access control, simplifies TLS management, and keeps application workloads isolated behind the load balancer.

The VPC architecture establishes the network foundation that enables the **Public Resources Architecture** to provide internet-facing services and egress capabilities, while the **Private Resources Architecture** hosts secure application workloads with controlled network access patterns.

---
### Public Resources Architecture
<div align="center">

![](./assets/Pseudo_Architectures/TF-INFRA_Public-Subnets.png)

*Figure 5: Public Subnet Resources Architecture - Internet-facing components including ALB capacities and Custom NAT instances*

***(Click on the architecture for a better full-screen view)***

</div>

The Public Resources Architecture manages internet-facing infrastructure components within strategically segmented public subnets, providing controlled ingress and egress capabilities while maintaining cost optimization through custom NAT instance implementations. This layer serves as the <ins>**network perimeter**</ins> that bridges external internet connectivity with internal private resources.

- **Subnet `A` (us-east-1a)**: `10.0.1.0/24` providing 251 (256 - 5 IPs reserved by AWS) usable IP addresses for high-availability resource deployment
- **Subnet `B` (us-east-1b)**: `10.0.2.0/24` providing 251 (256 - 5 IPs reserved by AWS) usable IP addresses for cross-AZ redundancy and load distribution
- **Reserved IP Space**: Each `/24` subnet reserves sufficient addressing for future expansion including bastion hosts, additional NAT instances, and potential migration to AWS Managed NAT Gateways

- **Multi-AZ Deployment**: ALB automatically provisions network interfaces across both public subnets for high availability and traffic distribution
- **Cross-Zone Load Balancing**: Enabled by default to ensure even traffic distribution across availability zones regardless of target distribution
- **Internet Gateway Integration**: Direct routing from ALB to IGW for inbound HTTP/HTTPS traffic from internet clients
- **Security Group Configuration**: ALB security group permits inbound traffic on ports 80 and 443 from `0.0.0.0/0` while restricting all other protocols

**`Custom NAT Instance Strategy:`** **ECS_CONTROL_PLANE_DNS_NAME** : The sole operational reason NAT instances are needed is to allow ECS agent and EC2-backed container instances in private subnets to reach the **regional ECS control plane**. You can avoid public internet egress by using <ins>VPC endpoints</ins>, but those endpoints typically incur additional charges and are not covered by the AWS Free Tier. For Free Tier–compatible, low-cost evaluations I therefore managed to setup outbound Internet access via **my own lightweight custom NAT EC2 instances** (one per AZ) instead of paid VPC endpoint alternatives.
- **Cost Optimization**: Deployed in each public subnet as `t2.micro` instances to maintain **AWS Free Tier eligibility** instead of using AWS Managed NAT Gateways (`$45/month` per gateway)
- **High Availability**: One NAT instance per availability zone providing redundant egress paths for private subnet resources
- **Instance Configuration**: Amazon Linux 2 AMI with custom User Data script handling IP forwarding, iptables rules, and source/destination check disabling

**`Network Routing & Traffic Flow:`**
- **Ingress Path**: Internet → IGW → ALB (Public Subnets) → Target Groups (Private Subnets)
- **Egress Path**: Private Resources → NAT Instances (Public Subnets) → IGW → Internet
- **Route Table Association**: Public subnets use route tables with `0.0.0.0/0` pointing to Internet Gateway for direct internet connectivity

**`Security & Access Control:`**
- **NAT Instance Security Groups**: Restrictive ingress rules allowing only traffic from private subnet CIDR ranges while permitting outbound internet access
- **Source/Destination Check**: Disabled on NAT instances to enable packet forwarding between private subnets and internet

The custom NAT instance implementation leverages a **User Data script** available in the codebase that configures the EC2 instances for NAT functionality, including network interface configuration, routing table setup, and traffic forwarding rules. Detailed technical implementation of these NAT instances will be covered in the Technical Implementation section.

### Private Resources Architecture
<div align="center">

![](./assets/Pseudo_Architectures/TF-INFRA_Private_Subnets.png)

*Figure 6: Private Subnet Resources Architecture - Internal ECS clusters and Associated ASGs (Auto-Scaling Groups)*

***(Click on the architecture for a better full-screen view)***

</div>

The Private Resources Architecture hosts the core application workloads within isolated private subnets, providing secure compute environments for containerized applications while maintaining strict network isolation from internet access. This layer implements <ins>**defense-in-depth security**</ins> through subnet isolation, security group controls, and NAT-based egress routing.

**`Private Subnet Distribution & CIDR Allocation:`**
- **Subnet `C` (us-east-1a)**: `10.0.3.0/24` providing 251 usable IP addresses for staging environment ECS cluster and associated resources
- **Subnet `D` (us-east-1b)**: `10.0.4.0/24` providing 251 usable IP addresses for production environment ECS cluster and cross-AZ redundancy
- **Reserved IP Space**: Each `/24` subnet reserves addressing capacity for database subnets, cache layers, and additional compute resources

**`ECS Clusters Architecture:`** 
This implementation uses ECS with EC2-backed container instances (not AWS Fargate). Choosing ECS EC2-based workloads provides an in‑depth infrastructural vision and full host-level control (AMIs, kernel tuning, custom NAT, networking, IAM roles, and node-level observability) needed for security hardening and operational transparency. It also reduces costs and preserves **AWS Free Tier eligibility** (t2.micro instances for low‑capacity ASGs and NAT) versus Fargate’s per-task pricing, which helps keep the factory affordable for evaluation and small-scale deployments.
### ECS Clusters & Auto Scaling

- **Staging Cluster**: Private subnet (us-east-1a). ASG: min `0`, desired `0`, max `2` (t2.micro for cost/Free Tier), as these staging instances are meant to be **EPHEMERAL**, and called only by the associated DAST stage and die afterwards.
- **Production Cluster**: Cross-AZ private subnets. ASG: min `1`, desired `1`, max `2` (t2.micro by default).
- **EC2 Launch Templates**: <ins>ECS-Optimized AMI</ins>, IAM instance profile, user-data, security groups, and CloudWatch/log agents for consistent hosts.
- **Scaling & Resilience**: Use target-tracking (CPU/memory) or step policies; enable ECS draining + ASG lifecycle hooks and health checks for graceful replacement.
- **Deployment Strategy**: Rolling updates (ASG + ECS) to maintain availability during instance replacements.
- **Operational Note**: Tune instance types and ASG bounds per workload; keep t2.micro for evaluation/Free Tier compatibility.

**`Database Layer Considerations:`**
For this implementation, **no database layer is deployed** to maintain simplicity as the factory will be tested with a lightweiht yet kind of vulnerable FastAPI application that uses mock JSON data. However, the architecture can support easy database integration through the following pattern:
- **Database Subnets**: Additional private subnets (`10.0.5.0/24`, `10.0.6.0/24`) reserved for RDS deployment across availability zones
- **Security Group Configuration**: Database security groups configured to accept connections only from ECS cluster security groups on database-specific ports
- **Read Replica Strategy**: Cross-AZ read replicas in the second availability zone for high availability and read scaling
- **Backup & Recovery**: Automated backup strategies with point-in-time recovery capabilities integrated with the existing monitoring infrastructure (+++ $$$)

The private subnet architecture ensures application workloads remain completely isolated from direct internet access while maintaining necessary connectivity for container orchestration, monitoring, and automated deployment processes through the CI/CD pipeline.

## AWS CloudFormation CI/CD Sub-Architecture
<div align="center">

![](./assets/Pseudo_Architectures/CF-RESOURCES.png)

*Figure 7: CloudFormation CI/CD Sub-Architecture - Complete pipeline infrastructure and AWS-native service orchestration*

***(Click on the architecture for a better full-screen view)***

</div>

The AWS CloudFormation CI/CD Sub-Architecture manages the pipeline orchestration layer of the DevSecOps Factory, handling AWS-native service integration and automated software delivery workflows. This layer implements <ins>**pipeline-as-code patterns**</ins> that provide secure, scalable CI/CD capabilities while maintaining seamless integration with the Terraform-managed infrastructure foundation.

**`Pipeline State Management:`**
- **CloudFormation Stack Management**: Declarative stack definitions with built-in drift detection and rollback capabilities for pipeline resource lifecycle
- **Parameter Integration**: Dynamic parameter injection from Terraform outputs enabling cross-IaC resource referencing and configuration
- **Resource Tagging**: Consistent tagging strategy for pipeline resources aligned with infrastructure layer for unified cost tracking and governance

**| <ins>Source Control Integration</ins>:**
- **AWS CodeConnections (ex-AWS CodeStar Connections)**: Secure remote git repo provider (GitLab, GitHub, Bitbucket, ...) integration providing webhook-based source code monitoring and automated pipeline triggering
- **Branch Strategy**: Multi-branch support enabling feature branch deployments and environment-specific pipeline execution
- **Source Artifact Management**: Automated source code packaging and versioning for downstream pipeline stage consumption

**| <ins>Build & Security Pipeline</ins>:**
- **AWS CodeBuild Projects**: Containerized build environments executing multi-stage security scanning including SAST, SCA, DAST, and compliance validation
- **Security by Design**: Embedded security tools (Snyk, OWASP ZAP, git-secrets, Clair) providing comprehensive vulnerability assessment across the software supply chain
- **Artifact Generation**: Container image building, security scanning, and artifact packaging for deployment to target environments
- **Secrets Management**: All sensitive values (secrets, API keys, endpoint URLs, tokens, etc.) are stored as SecureString parameters in <ins>AWS SSM Parameter Store</ins>. Pipeline and CodeBuild jobs access them via least‑privilege IAM roles and explicit parameter ARNs; no credentials are hardcoded in source or buildspecs.



**| <ins>Deployment Orchestration</ins>:**
- **Multi-Environment Deployment**: Automated <ins>staging</ins> & <ins>production</ins> deployment workflows with environment-specific configuration management
- **Blue/Green Strategy**: Zero-downtime deployment implementation through ECS service coordination and ALB target group management
- **Approval Gate**: Manual approval controls for production deployments ensuring human oversight for critical environment changes

**| <ins>Monitoring & Notifications</ins>:**
- **AWS EventBridge Integration**: Event-driven pipeline monitoring and automated incident response workflows
- **AWS CloudWatch Integration**: Comprehensive pipeline metrics, logging, and alerting across all stages and environments
- **AWS SNS Multi-topic Notifications**: Multi-topic notification distribution for pipeline status, security findings, and operational alerts

<br/>

Next, I will dive deeper into each section (***<ins>AWS CodePipeline Architecture</ins>*** & ***<ins>Pipeline Integration Workflow</ins>***)

### CodePipeline Architecture
<div align="center">

![](./assets/Pseudo_Architectures/CF-RESOURCES_codePipeline.png)

*Figure 8: CodePipeline Architecture - Multi-stage pipeline with integrated security scanning and deployment automation*

***(Click on the architecture for a better full-screen view)***

</div>

The CodePipeline Architecture implements a <ins>**multi-stage security-first automated delivery pipeline**</ins> that orchestrates the complete software delivery lifecycle from source code commit to production deployment. This pipeline integrates comprehensive security scanning, quality assurance, and deployment automation while maintaining strict separation between staging and production environments.

**`Pipeline Stage Configuration:`**
- **Source Stage**: AWS CodeConnections integration for automated source code retrieval from GitHub repository with webhook-triggered pipeline execution
- **Build Stage**: Five parallel CodeBuild projects executing comprehensive security scanning and artifact generation
- **Deploy-Staging Stage**: Automated deployment to ephemeral staging environment ECS cluster for integration testing and DAST validation
- **Approval Stage**: Manual approval gate requiring human authorization before production deployment for critical environment protection
- **Deploy-Production Stage**: Blue/green deployment to production ECS cluster with zero-downtime service updates and traffic switching

**`AWS CodeBuild Project Architecture:`**
- **Secrets-Scanning Project**: Dedicated build environment executing git-secrets for credential leak detection and sensitive data exposure prevention, with code artifact generation and S3 archiving
- **SAST-Snyk Project**: Static Application Security Testing environment using Snyk for code vulnerability analysis combined with Docker image building and ECR repository pushing
- **SCA-Clair Project**: Software Composition Analysis using Clair for dependency vulnerability and image scanning assessment with automated staging environment deployment coordination
- **DAST-OWASP-ZAP Project**: Dynamic Application Security Testing environment executing OWASP ZAP penetration testing against live staging application endpoints with automated approval triggering on successful validation
- **Production-BlueGreen Project**: Blue/green deployment orchestration managing ECS service updates, ALB target group switching, and zero-downtime production releases

Each CodeBuild execution provisions ephemeral, fresh, isolated container environments, ensuring a clean build state and preventing artifact contamination between pipeline runs.

I will tackle in the next section the whole CI/CD workflow and the pipeline’s integration with all the related services in the CI/CD process.

### Pipeline Integration & Complete CI/CD Workflow
<div align="center">

![](./assets/Pseudo_Architectures/CF-RESOURCES_cicd.png)

*Figure 9: Pipeline Integration & CI/CD Workflow - End-to-end CI/CD workflow with security gates, artifact management & multi-environment deployment orchestration*

***(Click on the architecture for a better full-screen view)***

</div>

The full CI/CD Workflow implements a <ins>**comprehensive end-to-end DevSecOps automation**</ins> that orchestrates secure software delivery through integrated security scanning, artifact management, centralized monitoring, and automated notification systems. This workflow ensures security compliance at every stage while maintaining operational transparency and incident response capabilities.

**`End-to-End Sequential Workflow:`**

***<ins>Phase 1: Source Code Trigger & Artifact Preparation</ins>***

1. Developer commits code to the remote repository triggering AWS CodeConnections webhook
2. CodePipeline automatically initiated with source artifact packaging and S3 storage
3. Pipeline execution metadata (commit SHA, timestamp, branch) captured for traceability

***<ins>Phase 2: SAST Scanning & Build Process</ins>***

4. Secrets-Scanning Project executes git-secrets analysis for credential exposure detection with immediate pipeline termination on any secret detection
5. SAST-Snyk Project performs static code analysis and vulnerability assessment:
   - **Low/Medium Vulnerabilities**: Docker container image built and pushed to ECR repository
   - **High/Critical Vulnerabilities**: Docker container image NOT built NOR pushed to ECR repository  
   - **Lambda Function Triggered**: Security findings from Snyk analysis sent to Lambda Normalization Function regardless of severity level
6. Lambda Normalization Function triggered by Snyk security scan completion. It  archives results to S3 artifact store with standardized scale & naming conventions, transformes findings to AWS Security Hub ASFF format with severity classification, and publishes normalized findings to AWS Security Hub for centralized visibility
7. Pipeline continues to next stage regardless of Snyk findings severity

***<ins>Phase 4: Container Image Security Analysis & Staging Decision</ins>***

8. SCA-Clair Project pulls container image from ECR and performs comprehensive dependency vulnerability scanning
    - **Low/Medium Vulnerabilities**: Staging environment deployment initiated
    - **High/Critical Vulnerabilities**: Pipeline stage fails and terminates
    - **Lambda Function Triggered**: Clair security findings sent to Lambda Normalization Function regardless of outcome
9. Lambda Normalization Function processes Clair findings and publishes to Security Hub (same process as Snyk stage)
10. If Clair vulnerabilities are Low/Medium: ECS staging cluster Auto Scaling Group scales from 0 to 1 instance
11. ECS task definition updated with validated container image from ECR repository
12. Staging application deployed and health checks validated before DAST execution

***<ins>Phase 5: DAST Pentesting</ins>***

13. DAST-OWASP-ZAP Project executes comprehensive **pentesting** against live staging endpoints:
    - **Low/Medium Vulnerabilities**: Manual approval gate triggered via SNS topic notification
    - **High/Critical Vulnerabilities**: Pipeline stage fails and terminates
    - **Lambda Function Triggered**: OWASP ZAP findings sent to Lambda Normalization Function regardless of outcome
14. Lambda Normalization Function processes DAST results and publishes to Security Hub (same process as Snyk or Clair stages)
15. Staging ECS cluster automatically scales down to 0 instances

***<ins>Phase 6: Security Gate Validation & Manual Approval (Conditional)</ins>***

16. Security team reviews consolidated security findings in Security Hub dashboard if SNS manual approval notification is sent by the previous stage
17. Manual approval gate activated requiring human authorization for production deployment

***<ins>Phase 7: Production Blue/Green Deployment (Approval Required)</ins>***

18. Upon Manual Approval, Production-BlueGreen Project execution triggered
19. New ECS task definition created with security-validated container image (Blue environment)
20. Blue environment health checks validated before traffic switching
21. ALB target group gradually switched from Green to Blue environment
22. Zero-downtime deployment completed with old Green environment termination

***<ins>Phase 8: Post-Deployment Monitoring & Cleanup</ins>***

23. CloudWatch metrics and EventBridge events monitor deployment success

***<ins>Phase 9: Continuous Monitoring & Feedback Loop</ins>***

24. RASP tool (CNCF Falco) provides continuous runtime security monitoring for production workloads, especially the **<ins>Docker container runtime</ins>**
25. Security Hub maintains historical tracking of all vulnerability findings across pipeline executions and RASP.


This sequential workflow ensures comprehensive security validation at every stage while maintaining operational efficiency and cost optimization through ephemeral staging environments and automated cleanup processes.

The Factory's CI/CD Workflow creates a comprehensive DevSecOps automation that ensures enterprise-grade software delivery.

---

# Section 3: Technical Implementation Details & Operations

## Modular Terraform Approach

<div align="center">

![Terraform Modular Structure](./assets/terraform_modular_folder_structure.png)

*Figure 10: Terraform Modular Folder Structure - Organized module-based infrastructure provisioning*

</div>

The Terraform implementation follows a <ins>**modular architecture pattern**</ins> that promotes code reusability, maintainability, and separation of concerns across different infrastructure domains. This approach enables independent module development, testing, and deployment while maintaining consistent resource provisioning and configuration management.

The Terraform codebase containes a **root module** that orchestrates composition and cross‑module data flow among **3 specialized modules** : <ins>***`Network`***</ins>, <ins>***`Compute`***</ins> and <ins>***`Storage`***</ins>, with each module exposing structured outputs consumed by others via explicit variable passing and output‑to‑input mappings, since each module declares its own **VARIABLES**, **MAIN**, and **OUTPUTS** files.

The modular approach enables independent infrastructure layer management while maintaining integration points necessary for the hybrid IaC strategy with CloudFormation pipeline resources.

### Global Variables & Outputs
The root Terraform configuration establishes <ins>**centralized variable management**</ins> and <ins>global output coordination</ins> across all modules, providing consistent configuration and enabling seamless cross-module data flow. The root level manages global settings, module orchestration, and structured output generation for CloudFormation integration.

**`Root Configuration File`**:

The main configuration file orchestrates module composition and establishes provider configurations:

```hcl
##-- Network Module --##
module "network" {
  source             = "./network"
  project_name       = var.project_name
  availability_zones = var.availability_zones
}

#--------------------------------------#
##-- Compute Module (ECS EC2-based) --##
module "compute" {
  source = "./compute"
  project_name                  = var.project_name
  vpc_id                        = module.network.vpc_id
  public_subnet_ids             = module.network.public_subnet_ids
  private_subnet_ids            = module.network.private_subnet_ids
  prod_alb_security_group_id    = module.network.prod_alb_security_group_id
  prod_ecs_security_group_id    = module.network.prod_ecs_security_group_id
  staging_alb_security_group_id = module.network.staging_alb_security_group_id
  staging_ecs_security_group_id = module.network.staging_ecs_security_group_id
}

#----------------------#
##-- Storage Module --##
module "storage" {
  source       = "./storage"
  project_name = var.project_name
}
```
> **Check File**

> [terraform-manifests/main.tf](./terraform-manifests/main.tf)

<br/>

**`Global Variable Definitions`**:

Centralized variable management ensures consistent configuration across all modules:

```hcl
variable "project_name" {
  description     = "Project name for the AWS DevSecOps CI/CD Platform"
  type            = string
  default         = "devsecops-platform"
}

variable "aws_account_region" {
  description = "AWS region of the associated AWS root account"
  type        = string
  default     = "us-east-1"
}

variable "availability_zones" {
  description = "Main & Secondary Availability Zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

...
```
> **Check File**

> [terraform-manifests/global_variables.tf](./terraform-manifests/global_variables.tf)

<br/>


**`Structured Output Generation:`**

Root outputs aggregate module outputs for CloudFormation parameter injection:

```hcl
#- Networking Outputs -#
output "vpc_id" {
  description = "VPC identifier for CloudFormation integration"
  value       = module.network.vpc_id
}

output "private_subnet_ids" {
  description = "Private subnet identifiers for ECS cluster deployment"
  value       = module.network.private_subnet_ids
}

output "alb_dns_name" {
  description = "Application Load Balancer DNS name for application access"
  value       = module.network.alb_dns_name
}

#- Compute Infrastructure Outputs -#
output "ecs_cluster_staging_name" {
  description = "ECS staging cluster name for pipeline deployment"
  value       = module.compute.ecs_cluster_staging_name
}

output "ecs_cluster_production_name" {
  description = "ECS production cluster name for pipeline deployment"
  value       = module.compute.ecs_cluster_production_name
}

output "ecr_repository_uri" {
  description = "ECR repository URI for container image storage"
  value       = module.compute.ecr_repository_uri
}

...

```

> **Check File**

> [terraform-manifests/global_outputs.tf](./terraform-manifests/global_outputs.tf)

<br/>


The root config module establishes dependency relationships and data flow between modules through explicit output-to-input variable mapping, ensuring proper resource provisioning order and configuration consistency across the entire infrastructure stack.

### Network Module

The Network Module implements the foundational VPC architecture with strategic subnet segmentation, security group configuration, and custom NAT instance deployment. This module establishes the network perimeter and traffic control policies that enable secure multi-tier application deployment with least-privilege access controls.

#### Security Groups & Firewalling Strategy

The security group implementation follows a <ins>**zero-trust network model**</ins> with explicit allow rules and default-deny policies. Each security group enforces least-privilege access with specific protocol and port restrictions.

**`ALB Security Group Configuration:`**

```hcl
##-- Application Load Balancer Security Group --##
resource "aws_security_group" "alb_sg" {
  name_prefix = "${var.project_name}-alb-sg"
  vpc_id      = aws_vpc.main.id

  #- HTTP Inbound from Internet -#
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP access from internet"
  }

  #- HTTPS Inbound from Internet -#
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS access from internet"
  }

  #- All Outbound Traffic -#
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }

  tags = {
    Name = "${var.project_name}-alb-sg"
  }
}
```

**`ECS Security Group Configuration:`**

```hcl
##-- ECS Security Group - Production --##
resource "aws_security_group" "ecs_prod_sg" {
  name_prefix = "${var.project_name}-ecs-prod-sg"
  vpc_id      = aws_vpc.main.id

  #- HTTP from ALB Security Group Only -#
  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
    description     = "HTTP from ALB only"
  }

  #- HTTPS from ALB Security Group Only -#
  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
    description     = "HTTPS from ALB only"
  }

  ##-- All Outbound Traffic (Reginal ECS Control Plane access constraint) --##
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }

  tags = {
    Name = "${var.project_name}-ecs-prod-sg"
  }
}
```

**`NAT Instance Security Group Configuration:`**

```hcl
##-- NAT Instance Security Group --##
resource "aws_security_group" "nat_sg" {
  name_prefix = "${var.project_name}-nat-sg"
  vpc_id      = aws_vpc.main.id

  #- HTTP from Private Subnets Only -#
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [aws_subnet.private_subnet_a.cidr_block, aws_subnet.private_subnet_b.cidr_block]
    description = "HTTP from private subnets"
  }

  #- HTTPS from Private Subnets Only -#
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_subnet.private_subnet_a.cidr_block, aws_subnet.private_subnet_b.cidr_block]
    description = "HTTPS from private subnets"
  }

  #- All Outbound Traffic -#
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }

  tags = {
    Name = "${var.project_name}-nat-sg"
  }
}
```

> **Check File**

> [terraform-manifests/network/main.tf](./terraform-manifests/network/main.tf)

<br/>

**`Least-Privilege Access Strategy:`**

The security group architecture implements **defense-in-depth** through layered access controls:

- **`Internet Access`**: Only ALB security group permits inbound traffic from `0.0.0.0/0` on ports 80/443
- **Backend Protection**: ECS security groups **exclusively** accept traffic from ALB security group, preventing direct internet access
- **`Staging Cluster Exclusiveness`**: The staging cluster is only accessible by the **CodeBuild Security Group**.
- **`NAT Isolation`**: NAT instances only accept traffic from private subnet CIDR blocks, ensuring controlled egress
- **`Source/Destination Validation`**: Security groups reference other security groups rather than IP ranges, providing dynamic access control
- **`Protocol Restriction`**: Explicit protocol and port definitions prevent unauthorized service access

This strategy ensures that **no private resource can receive direct internet traffic**, while maintaining necessary connectivity for application functionality and system updates.

#### Custom NAT EC2 instances

The custom NAT instance implementation provides **cost-optimized internet egress** for private subnet resources while maintaining AWS Free Tier compatibility. These instances replace AWS Managed NAT Gateways, reducing monthly costs from `$90` to `$0` for small-scale deployments.


**`NAT Instance User Data Script`**:

The User Data script configures EC2 instances for NAT functionality with IP forwarding and iptables rules:

```bash
    #!/bin/bash
    sudo yum install iptables-services -y
    sudo systemctl enable iptables
    sudo systemctl start iptables

    #- Enable IP forwarding -#
    echo "net.ipv4.ip_forward=1" >> /etc/sysctl.d/custom-ip-forwarding.conf
    sudo sysctl -p /etc/sysctl.d/custom-ip-forwarding.conf

    PUBLIC_IFACE=$(ip route | awk '/default/ {print $5}')

    sudo /sbin/iptables -t nat -A POSTROUTING -o $PUBLIC_IFACE -j MASQUERADE
    sudo /sbin/iptables -F FORWARD
    sudo service iptables save
```

> **Check File**

> [terraform-manifests/network/main.tf](./terraform-manifests/network/main.tf)

<br/>

The custom NAT implementation provides **high availability** through multi-AZ deployment, **cost optimization** through Free Tier-eligible t2.micro instances, and **security hardening** through restricted security group rules and automated configuration management.

### Compute Module

The Compute Module implements the containerized application infrastructure with separate staging and production ECS EC2-Based clusters, providing secure, scalable compute environments for application workloads. 

This module manages the complete container orchestration lifecycle from cluster provisioning through task execution and scaling policies.

#### ECS Staging & Production Clusters

The ECS clusters architecture implements **environment-specific isolation** with distinct clusters for staging and production workloads, each optimized for their respective operational requirements and scaling characteristics.

> **Check File**

> [terraform-manifests/compute/main.tf](./terraform-manifests/compute/main.tf)

<br/>

**`Cluster Characteristics:`**
- **Staging Cluster**: Ephemeral operation with auto-scaling from 0-2 instances, single AZ deployment for cost optimization
- **Production Cluster**: Persistent operation with 1-3 instances, multi-AZ deployment for high availability
- **Container Insights**: Enabled on both clusters for comprehensive monitoring and observability
- **Capacity Providers**: Custom EC2 capacity providers with managed scaling for cost optimization

#### ECS EC2 Launch Template

The launch template implementation provides **standardized, immutable infrastructure** for ECS container instances with integrated security hardening, monitoring, and operational tooling.

**`Launch Template Configuration:`**

```hcl
##-- Launch Template for ECS Instances --##
resource "aws_launch_template" "ecs_production_lt" {
  name_prefix   = "${var.project_name}-production-lt"
  image_id      = data.aws_ami.ecs_optimized.id
  instance_type = var.instance_type
  key_name      = var.key_pair_name

  vpc_security_group_ids = [var.prod_ecs_security_group_id]

  iam_instance_profile {
    name = aws_iam_instance_profile.ecs_instance_profile.name
  }

  user_data = base64encode(templatefile("${path.module}/user_data.tpl", {
    cluster_name = aws_ecs_cluster.production.name
  }))

  ...

  monitoring {
    enabled = true
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${var.project_name}-production-ecs-instance"
      Environment = "production"
    }
  }
}
```

**`User Data Script:`**

```bash
#!/bin/bash
echo "ECS_CLUSTER=${cluster_name}" >> /etc/ecs/ecs.config
echo "ECS_ENABLE_TASK_IAM_ROLE=true" >> /etc/ecs/ecs.config
echo "ECS_ENABLE_TASK_IAM_ROLE_NETWORK_HOST=true" >> /etc/ecs/ecs.config

#- Install CloudWatch agent -#
yum install -y amazon-cloudwatch-agent

#- Start ECS agent -#
start ecs
```

> **Check File & User Data script**

> [terraform-manifests/compute/main.tf](./terraform-manifests/compute/main.tf)
>
> [terraform-manifests/compute/user_data.tpl](./terraform-manifests/compute/user_data.tpl)

<br/>

#### Task Definitions

The task definition implementation provides ***comprehensive container orchestration*** with integrated security monitoring through **<ins>CNCF Falco sidecar</ins>** patterns, ensuring runtime application security protection alongside application containers.

**`Production Task Definition with Falco Sidecar:`**

```hcl
##-- ECS Task Definition - Production with Falco RASP --##
resource "aws_ecs_task_definition" "prod" {
  family                   = "${var.project_name}-app-prod-task-def"
  network_mode             = "bridge"
  requires_compatibilities = ["EC2"]
  cpu                      = "256"
  memory                   = "400"
  
  container_definitions    = jsonencode([
    {
      name      = "${var.project_name}-app-prod-container"
      image     = "nginx:alpine"
      essential = true
      memory    = 256
      memoryReservation = 128
      portMappings = [{ 
        containerPort = 80, 
        hostPort = 0
      }]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/aws/ecs/${var.project_name}/production"
          "awslogs-region"        = "us-east-1"
          "awslogs-stream-prefix" = "ecs"
        }
      }
    },
    {
      name      = "falco"
      image     = "falcosecurity/falco:latest"
      essential = false
      memory    = 128
      memoryReservation = 64
      privileged = true
      mountPoints = [
        { sourceVolume = "docker-socket", containerPath = "/host/var/run/docker.sock" }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "${var.project_name}-falco-logs"
          "awslogs-region"        = "us-east-1"
        }
      }
    }
  ])
  
  volume {
    name = "docker-socket"
    host_path = "/var/run/docker.sock"
  }
}
```

> **Check File**

> [terraform-manifests/compute/main.tf](./terraform-manifests/compute/main.tf)

<br/>

**`Falco Sidecar Integration:`**
- **Runtime Security Monitoring**: Falco provides real-time detection of anomalous behavior and security threats during application execution
- **Sidecar Pattern**: Falco runs as a separate container within the same task, sharing the network namespace and monitoring the application container
- **Host System Access**: Privileged container with host filesystem mounts for comprehensive system call monitoring
- **CloudWatch Integration**: Falco security alerts forwarded to CloudWatch for centralized monitoring and alerting

### Storage Module

The Storage Module implements **centralized artifact management** and **Lambda function packaging** infrastructure, providing secure, encrypted S3 storage for CI/CD pipeline artifacts, security scan results, and Lambda deployment packages, in the form of <ins>**2 S3 Buckets**</ins>.

#### S3 Artifact Store

The S3 artifact store provides **comprehensive pipeline artifact management** with encryption for secure storage of build artifacts, security scan results, and deployment packages.

> **Check File**

> [terraform-manifests/storage/main.tf](./terraform-manifests/storage/main.tf)

<br/>

#### S3 Lambda Packaging Bucket

The Lambda packaging S3 bucket provides **specialized storage** for Lambda function deployment package `(.zip)`.

> **Check File**

> [terraform-manifests/storage/main.tf](./terraform-manifests/storage/main.tf)

<br/>

**`Lambda Packaging Features:`**
- **Automated Packaging**: Lambda ZIP package automatically uploaded during Terraform deployment
- **Service Integration**: Bucket policies enabling secure access from Lambda and CodePipeline services
- **Package Management**: Automated Lambda function package updates through CI/CD pipeline
- **Security Controls**: Restrictive bucket policies ensuring least-privilege access to Lambda packages


The Storage Module provides the foundational storage infrastructure that enables secure artifact management, Lambda function deployment, and comprehensive audit logging throughout the DevSecOps pipeline lifecycle.

## AWS CloudFormation template

The AWS CloudFormation template implements the complete CI/CD pipeline orchestration and security integration components of the DevSecOps Factory. This template can be deployed as a **standalone solution** with custom parameter values, but <ins>**is designed for automated deployment through the hybrid IaC approach**</ins> where Terraform outputs <ins>are passed</ins> as CloudFormation parameters.

### CI/CD Workflow

The CI/CD workflow orchestrates the complete software delivery lifecycle through integrated AWS services, providing automated security scanning, compliance validation, and deployment automation. The workflow implements a **security-gate pattern** where each stage validates security posture before progression.

**`CloudFormation Template Overview:`**

```yaml
AWSTemplateFormatVersion: "2010-09-09"

Description: >
  A fully automated AWS DevSecOps Hybrid CI/CD platform that provisions secure platform infrastructure using
  Terraform and orchestrates a multi-stage CodePipeline via CloudFormation, integrating code
  and container security scanning, centralized artifact storage with encryption, compliance monitoring,
  and automated deployment to ECS clusters.

Parameters:
  #- Terraform Platform/Infra Related Parameters -#
  StagingECSCluster:
    Description: ECS Cluster for Staging
    Type: String
  
  ProdECSCluster:
    Description: ECS Cluster for Production
    Type: String

  EcrRegistryName:
    Description: ECR Registry Name
    Type: String
  
  #- Security Scanning Parameters -#
  SnykAPIKey:
    Description: Snyk API Key
    Type: String
    NoEcho: true
  
  AppURLForDAST:
    Description: Application URL to run the Dynamic Application Security Testing
    Type: String

  #- Git Repository Configuration -#
  GitProviderType:
    Description: Git Repository provider type
    Type: String
    AllowedValues:
      - GitHub
      - GitLab
      - Bitbucket

...
```
> **Check File**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)

<br/>

#### AWS CodeConnections Connection

The AWS CodeConnections Connection *(formerly CodeStar Connections)* establishes secure integration between the CI/CD pipeline and the external Git repository, providing webhook-based automation and secure credential management without exposing repository credentials.

**`CodeConnections Resource Configuration:`**

```yaml
CodeConnection:
  Type: AWS::CodeStarConnections::Connection
  Properties:
    ConnectionName: !Sub ${AWS::StackName}-conn
    ProviderType: !Ref GitProviderType
    Tags:
      - Key: pipeline-name
        Value: !Sub ${AWS::StackName}-pipeline
```

**`CloudWatch Event Integration:`**

```yaml
CloudWatchEventRule:
  Type: 'AWS::Events::Rule'
  Properties:
    EventPattern:
      source:
        - aws.codestar-connections
      detail-type:
        - CodeStar Source Connection State Change
      detail:
        state:
          - AVAILABLE
    Targets:
      - Arn: !Sub 'arn:${AWS::Partition}:codepipeline:${AWS::Region}:${AWS::AccountId}:${AWS::StackName}-pipeline'
        RoleArn: !GetAtt CloudWatchEventRole.Arn
        Id: codepipeline-AppPipeline
```

> **Check File**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)

<br/>

**`Manual Authorization Requirement:`**

When the CloudFormation template is deployed, the CodeConnections connection will be in a **PENDING** state and requires **ONE-TIME manual authorization** through the AWS Console:

1. Navigate to **AWS CodePipeline → Settings → Connections** in the AWS Console
2. Locate the connection.
3. Click **Update pending connection** and authorize access to your Git repository
4. Once authorized, the connection status changes to **AVAILABLE** and triggers automatic pipeline execution

This manual step ensures secure repository access while maintaining automated pipeline triggering for subsequent code commits.

#### Normalization & Aggregation Lambda Function

The Lambda function serves as the **security data processing engine**, normalizing security scan outputs from multiple tools into standardized AWS Security Hub findings format (ASFF) and providing centralized vulnerability correlation and deduplication.

<div align="center">

![](./assets/lambda_function_layout.png)

*Figure 11: Modular Lambda Normalization & Aggregation function - Custom logging, Isolated Security Hub Client module and Config separation with custom scale*

</div>

[TO BE CONTINUED]
**`Lambda Function Architecture:`**

The Lambda function implements a **modular, scalable security data processing architecture** with dedicated modules for each security tool integration. The function structure includes:

1. **`lambda_handler.py`**: Main handler orchestrating security scan result processing and AWS Security Hub integration
2. **`securityhub_client.py`**: Dedicated AWS Security Hub client with ASFF (AWS Security Finding Format) transformation capabilities
3. **`config.json`**: Centralized configuration management for security finding severity mapping and tool-specific settings
**4. `config_manager.py`**: Configuration management module that loads and validates settings from the `config.json` file.
**5. `report_processor.py`**: The <ins>main security report processing **engine**</ins> that handles the parsing, validation, and standardization of security scan outputs from different tools (Snyk, Clair, OWASP ZAP). This module extracts vulnerability data, normalizes finding formats, applies severity classifications, and prepares structured data for ASFF transformation and Security Hub ingestion.
6. **`logger_config.py`**: Custom logging module providing structured logging.

The Lambda architecture enables **extensible security tool integration** through standardized input processing, consistent ASFF output formatting, and centralized error handling across all supported security scanning tools (Snyk, Clair & OWASP ZAP).

**`Lambda Function Configuration:`**

```yaml
LambdaSHImport:
  Type: 'AWS::Lambda::Function'
  Properties:
    FunctionName: ImportToSecurityHub
    Handler: !Ref LambdaHandler
    Role: !GetAtt LambdaExecutionRole.Arn
    Runtime: python3.9
    Code:
      S3Bucket: !Ref LambdaS3Bucket
      S3Key: !Ref LambdaS3Key
    Environment:
      Variables:
        S3_ARTIFACT_BUCKET_NAME: !Ref PipelineArtifactS3Bucket
        AWS_PARTITION: aws
    Timeout: 10
```

> **Check File**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)

<br/>

**`Security Data Processing Workflow:`**

The Lambda function is triggered by each CodeBuild security scanning stage and performs:
- **Data Extraction**: Retrieves security scan results
- **Format Normalization**: Converts tool-specific outputs (Snyk, Clair, OWASP ZAP) to ASFF format
- **Severity Classification**: Standardizes vulnerability severity levels across different tools
- **Data Archival**: Pushes every aggregated finding into the artifact store
- **Security Hub Integration**: Publishes normalized findings to AWS Security Hub for centralized visibility

#### AWS CodeBuild Projects

The CodeBuild projects implement the **multi-stage security scanning pipeline** with five specialized build environments, each optimized for specific security analysis tasks. 

> [!IMPORTANT]
> All CodeBuild projects reference buildspec files **<ins>must be located in the application codebase under the `buildspecs/` directory</ins>**.

**`Buildspec File Structure Requirements:`**

```
application-repository/
├── buildspecs/
│   ├── 1-secretsanalysis-buildspec.yml
│   ├── 2-snyk-sast-buildspec.yml
│   ├── 3-clair-sca-staging-buildspec.yml
│   ├── 4-owasp-zap-dast-buildspec.yml
│   └── 5-prod-bluegreen-buildspec.yml
├── src/
├── Dockerfile
└── ...
```

> **Check Files**

> [buildspec/1-secretsanalysis-buildspec.yml](./buildspec/1-secretsanalysis-buildspec.yml)
>
> [buildspec/2-snyk-sast-buildspec.yml](./buildspec/2-snyk-sast-buildspec.yml)
>
> [buildspec/3-clair-sca-staging-buildspec.yml](./buildspec/3-clair-sca-staging-buildspec.yml)
>
> [buildspec/4-owasp-zap-dast-buildspec.yml](./buildspec/4-owasp-zap-dast-buildspec.yml)
>
> [buildspec/5-prod-bluegreen-buildspec.yml](./buildspec/5-prod-bluegreen-buildspec.yml)

<br/>

#### Blue/Green Deployment Strategy
The Blue/Green deployment strategy provides **zero-downtime production releases** through ECS service coordination and Application Load Balancer target group management, ensuring seamless traffic switching and automatic rollback capabilities.

**`Production Deployment CodeBuild Project:`**

```yaml
ProductionBlueGreenBuildProject:
  Type: AWS::CodeBuild::Project
  Properties:
    Description: Production Blue/Green Deployment using ECS Service Updates
    Environment:
      ComputeType: BUILD_GENERAL1_SMALL
      Image: aws/codebuild/standard:5.0
      Type: LINUX_CONTAINER
      EnvironmentVariables:
      - Name: ECR_REPO_URI
        Value: !Sub ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/${EcrRegistryName}
      - Name: ECS_CLUSTER_NAME
        Value: !Ref ProdECSCluster
      - Name: ECS_TASK_DEFINITION
        Value: !Ref ProdECSTaskDefinition
      - Name: ECS_SERVICE_NAME
        Value: !Ref ProdECSService
    VpcConfig:
      VpcId: !Ref VpcId
      Subnets: !Ref PrivateSubnetIds 
      SecurityGroupIds: 
        - !Ref CodeBuildSecurityGroupId
    Source:
      Type: CODEPIPELINE
      BuildSpec: buildspecs/5-prod-bluegreen-buildspec.yml
```

> **Check Files**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)
>
> [buildspec/5-prod-bluegreen-buildspec.yml](./buildspec/5-prod-bluegreen-buildspec.yml)

<br/>

This approach ensures **production stability** while enabling rapid deployment cycles and immediate rollback capabilities for critical production environments.

### Security & Compliance

The Security & Compliance implementation establishes comprehensive security controls and compliance monitoring throughout the DevSecOps Factory, ensuring that security is embedded at every stage of the software delivery lifecycle. This section implements **defense-in-depth security** through multiple security tools, encryption standards, and compliance frameworks.

#### SSM Parameter Store

AWS Systems Manager Parameter Store provides **centralized, encrypted secret management** for all sensitive configuration data used throughout the CI/CD pipeline. Every secret passed to the CloudFormation stack parameters is automatically mapped to corresponding SSM SecureString parameters.

**`Parameter Mapping Strategy:`**

```yaml
#- SSM Parameters for Secure Secrets/Config Storage -#
SSMSnykAPIKey:
  Type: 'AWS::SSM::Parameter'
  Properties:
    Name: !Sub ${AWS::StackName}-Snyk-API-Key
    Type: String 
    Value: !Ref SnykAPIKey

SSMAppURL:
  Type: 'AWS::SSM::Parameter'
  Properties:
    Name: !Sub ${AWS::StackName}-App-URL
    Type: String
    Value: !Ref AppURLForDAST

SSMDockerHubPassword:
  Type: 'AWS::SSM::Parameter'
  Properties:
    Name: !Sub ${AWS::StackName}-DockerHub-Password
    Type: String
    Value: !Ref DockerHubPassword
```

**`CodeBuild Parameter Store Integration:`**

```yaml
EnvironmentVariables: 
- Name: SNYK_API_KEY
  Type: PARAMETER_STORE
  Value: !Ref SSMSnykAPIKey
- Name: APPLICATION_URL
  Type: PARAMETER_STORE
  Value: !Ref SSMAppURL
- Name: DOCKERHUB_PASSWORD
  Type: PARAMETER_STORE
  Value: !Ref SSMDockerHubPassword
```

> **Check File**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)

<br/>

#### Encryption & KMS

The platform implements **comprehensive encryption-at-rest** using a single AWS KMS symmetric key for all pipeline-related encryption, ensuring consistent security policies and simplified key management.

**`Pipeline KMS Key Configuration:`**

```yaml
PipelineKMSKey:
  Type: AWS::KMS::Key
  Properties: 
    Description: KMS Key for General Pipeline-Related Encryption
    Enabled: true
    EnableKeyRotation: true
    KeyPolicy: 
      Version: '2012-10-17'
      Statement:
      - Effect: Allow
        Principal:
          AWS: !Sub 'arn:${AWS::Partition}:iam::${AWS::AccountId}:root'
        Action: 
          - 'kms:Create*'
          - 'kms:Describe*'
          # ...additional admin permissions...
        Resource: '*'
      - Effect: Allow
        Principal:
          Service: 
            - codepipeline.amazonaws.com
            - codebuild.amazonaws.com
            - !Sub logs.${AWS::Region}.amazonaws.com
        Action:
        - 'kms:Encrypt'
        - 'kms:Decrypt'
        - 'kms:ReEncrypt*'
        - 'kms:GenerateDataKey*'
        Resource: '*'
```

**`Encryption Scope:`**
- **S3 Artifact Store**: All pipeline artifacts encrypted using the KMS key
- **CodeBuild Projects**: All projects configured with KMS encryption

> **Check File**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)

<br/>

#### Secrets Scanning (git-secrets)

Git-secrets implementation provides **credential leak detection** through pattern-based scanning of source code, preventing accidental exposure of sensitive information in the codebase.

**`Key Security Commands:`**

```bash
...

#- Install and configure git-secrets -#
git secrets --register-aws
git secrets --install

#- Custom pattern detection for additional secrets -#
git secrets --add 'password\s*=\s*.+'
git secrets --add 'secret\s*=\s*.+'
git secrets --add 'api[_-]?key\s*=\s*.+'

git secrets --scan --recursive .

...
```

> **Check File**

> [buildspecs/1-secretsanalysis-buildspec.yml](./buildspecs/1-secretsanalysis-buildspec.yml)

<br/>

#### SAST - Static Application Security Analysis (Snyk)

Snyk SAST integration provides **comprehensive static code analysis** for vulnerability detection, license compliance, and security policy enforcement during the build process.

**`Key Security Commands:`**

```bash
...

snyk config set api=$SNYK_API_KEY


#- Perform code vulnerability scanning -#
snyk code test --severity-threshold=high

#- Container image vulnerability scanning -#
snyk container test $ECR_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION --file=$DOCKERFILE_NAME --json --severity-threshold=high > snyk-results.json || echo "Snyk SAST Scan completed with findings"

...

```

> **Check File**

> [buildspecs/2-snyk-sast-buildspec.yml](./buildspecs/2-snyk-sast-buildspec.yml)

<br/>

#### SCA - Software Composition Analysis (Clair)

Clair SCA implementation provides **container image vulnerability scanning** through comprehensive analysis of container layers and installed packages, integrated with ECR's native vulnerability scanning capabilities.

**`Key Security Commands:`**

```bash
...

# ECR vulnerability scan initiation
aws ecr start-image-scan --repository-name $ECR_REPO_NAME

# Retrieve scan results
aws ecr describe-image-scan-findings --repository-name $ECR_REPO_NAME

...

```

> **Check File**

> [buildspecs/3-clair-sca-staging-buildspec.yml](./buildspecs/3-clair-sca-staging-buildspec.yml)

<br/>

#### DAST - Dynamic Application Security Analysis (OWASP ZAP)

OWASP ZAP DAST implementation provides **runtime security testing** against live staging applications, performing comprehensive penetration testing to identify vulnerabilities in running applications.

**`Key Security Commands:`**

```bash
...

#- Start ZAP daemon -#
/opt/zap/zap.sh -daemon -host 0.0.0.0 -port 8080 -config api.disablekey=true -config api.addrs.addr.name=.* -config api.addrs.addr.regex=true > /tmp/zap.log 2>&1 &


#- Spider the application -#
spider_response=$(curl -s "http://localhost:8080/JSON/spider/action/scan/?url=http://$APPLICATION_URL")
echo "Spider API Response: $spider_response"

...

while [ "$stat" != "100" ] && [ $counter -lt $timeout ]; do
  stat=$(curl "http://localhost:8080/JSON/spider/view/status/?scanId=$spider_scanid" | jq -r '.status');
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] Spider scan status: $stat%"
  if [ "$stat" = "100" ]; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] Spider scan completed successfully."
    break
  fi
  sleep 10;
  counter=$((counter + 10))
done

...

#- Active security scanning -#
active_response=$(curl -s "http://localhost:8080/JSON/ascan/action/scan/?url=http://$APPLICATION_URL")
echo "Active Scan API Response: $active_response"

...Same Loop

...

```

> **Check File**

> [buildspecs/4-owasp-zap-dast-buildspec.yml](./buildspecs/4-owasp-zap-dast-buildspec.yml)

<br/>

#### RASP - Runtime Application Security Protection (CNCF Falco)
CNCF Falco provides **runtime security monitoring** for containerized applications, detecting anomalous behavior and security threats during application execution. Falco is deployed using the **<ins>sidecar pattern<ins>** alongside application containers in ECS task definitions.

**`Sidecar Deployment Pattern:`**

Falco is implemented as a sidecar container within ECS task definitions, providing real-time security monitoring without modifying application code. The detailed Falco sidecar configuration and integration with application containers is covered in the [Compute Module - Task Definitions](#task-definitions) section.

The RASP implementation ensures **continuous security monitoring** throughout the application lifecycle, providing real-time threat detection and incident response capabilities for production workloads.

### Event-Driven Architecture

The Event-Driven Architecture implements **comprehensive event monitoring and automated response capabilities** throughout the DevSecOps Factory, enabling real-time operational awareness and incident response across all pipeline stages and infrastructure components.

#### AWS EventBridge Rules

EventBridge provides **centralized event routing** for pipeline state changes, infrastructure events, and security findings, enabling automated workflows and operational responses.

**`Pipeline State Monitoring:`**

```yaml
CloudWatchPipelineEventRule:
  Type: 'AWS::Events::Rule'
  Properties:
    EventPattern:
      source:
        - aws.codepipeline
      detail-type:
        - CodePipeline Stage Execution State Change
    Targets:
      - Arn: !Ref PipelineTopic
        Id: "PipelineNotifications"
```

> **Check File**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)

<br/>

#### AWS CloudWatch Events

CloudWatch Events integration provides **automated pipeline triggering** and cross-service event coordination, ensuring seamless workflow automation across AWS services.

**`Automated Pipeline Execution:`**
- **CodeConnections Integration**: Automatic pipeline triggering on repository state changes
- **Cross-Service Coordination**: Event-driven communication between CodePipeline, CodeBuild, and ECS services

#### SNS Topics & Subscriptions

The platform implements a **multi-topic pub/sub architecture** for comprehensive notification distribution across different operational contexts and stakeholder groups.

**`Multi-Topic Architecture:`**

```yaml
ManualApprovalTopic:
  Type: AWS::SNS::Topic
  Properties:
    DisplayName: PipelineManualApprovalTopic
    Subscription: 
    - Endpoint: !Ref PipelineManualApproverMail
      Protocol: "email"

PipelineTopic:
  Type: AWS::SNS::Topic
  Properties:
    DisplayName: PipelineStageChangeNotificationTopic
    Subscription: 
    - Endpoint: !Ref PipelineNotificationMail
      Protocol: "email"

CloudTrailTopic:
  Type: AWS::SNS::Topic
  Properties:
    DisplayName: CloudTrailNotificationTopic
    Subscription: 
    - Endpoint: !Ref PipelineNotificationMail
      Protocol: "email"
...

```

> **Check File**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)

<br/>

- **`Manual Approval Topic`**: Dedicated notifications for production deployment approvals requiring human intervention
- **`Pipeline State Topic`**: Operational notifications for pipeline stage changes, build status, and deployment progress
- **`CloudTrail Audit Topic`**: Security and compliance notifications for infrastructure changes and API activity monitoring

### Monitoring & Observability

The Monitoring & Observability implementation provides **comprehensive visibility** into all aspects of the DevSecOps Factory, from infrastructure performance to security posture and compliance status.

#### AWS CloudWatch Dedicated Log Groups

CloudWatch Log Groups implement **structured logging** across all pipeline components with dedicated log streams for different operational contexts and security analysis.

**`Pipeline-Specific Log Groups:`**

```yaml
PipelineCloudWatchLogGroup:
  Type: AWS::Logs::LogGroup
  Properties: 
    LogGroupName: !Sub ${AWS::StackName}-pipeline-logs
    RetentionInDays: 7

TrailLogGroup:
  Type: 'AWS::Logs::LogGroup'
  Properties:
    LogGroupName: !Sub ${AWS::StackName}-cloudtrail-logs
    RetentionInDays: 14
```

> **Check File**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)

<br/>

#### AWS CloudTrail & AWS Config

CloudTrail and Config provide **comprehensive compliance monitoring** and **configuration drift detection** across all infrastructure and pipeline resources.

> **Check File**

> [cloudformation/codepipeline.yaml](./cloudformation/codepipeline.yaml)

<br/>

The Event-Driven Architecture and Monitoring & Observability systems work together to provide **real-time operational intelligence** and **proactive incident response**, ensuring the DevSecOps Factory maintains optimal performance, security, and compliance posture.

---

# Section 4: Deployment & Configuration Guide

## Deployment Scripts

The AWS DevSecOps Hybrid CI/CD Platform provides **automated deployment scripts** that orchestrate the complete platform provisioning with a single command. These scripts handle the complex integration between Terraform infrastructure deployment and CloudFormation pipeline provisioning, automatically passing Terraform outputs as CloudFormation parameters.

The deployment automation implements a **sequential orchestration pattern** that ensures proper dependency resolution and resource integration:


1. **Prerequisites Validation**: Comprehensive validation of required tools, credentials, and configuration
2. **Lambda Package Creation**: Automated packaging of the Security Hub Lambda function
3. **Infrastructure Deployment**: Terraform infrastructure provisioning with state management
4. **Output Extraction**: Automated capture and processing of Terraform outputs
5. **Pipeline Deployment**: CloudFormation stack deployment with parameter injection
6. **Integration Verification**: Post-deployment validation and status reporting

***Cross-Platform Script Support:***

- **`deploy.sh`** (Bash): Primary deployment script supporting Linux, macOS, and Windows (Git Bash/WSL)
- **`deploy.ps1`** (PowerShell): Windows-native PowerShell implementation

> **Check Scripts**

> [Bash Script](./scripts/deploy.sh)
>
> [PowerShell Script](./scripts/deploy.ps1)

<br/>

> [!WARNING]
> The PowerShell script is not stable and in Dev Mode. Please stick to deploying using the `Bash script` also for non Linux devices by using a bash shell, it has built-in OS detection.

- **One-Command Deployment**: Complete platform deployment with `./deploy.sh`
- **Automated Rollback**: Complete cleanup with `--rollback-deployment` flag
- **Hybrid IaC Integration**: Automatic Terraform output extraction and CloudFormation parameter injection
- **Error Handling**: Comprehensive error detection and rollback capabilities
- **Progress Logging**: Detailed deployment progress with colored output and timestamps

**`Command Examples:`**

```bash
##- Show help and available options -##
./deploy.sh --help

##- Full deployment with new infrastructure -##
./deploy.sh


#- Complete rollback and cleanup -#
./deploy.sh --rollback-deployment

#- Deploy only CI/CD pipeline to existing infrastructure -#
./deploy.sh --skip-infrastructure

```

The deployment scripts eliminate manual parameter mapping between IaC tools, ensuring consistent deployments and reducing configuration errors through automated cross-tool integration.

## Environment Configuration Reference
The platform configuration is managed through a **centralized environment file** (`.env`) that contains all required parameters for deployment and operation. This approach ensures consistent configuration across deployment scripts and infrastructure provisioning.

The Environment File Structure is mentionned in the [.env.template](./.env.template) file, and is the following:

```bash
#-- Remote Git Repository Configuration (For AWS CodeConnections Connection) --#
GIT_PROVIDER_TYPE=""
FULL_GIT_REPOSITORY_ID=""
BRANCH_NAME=""

#-- Snyk API Key --#
SNYK_API_KEY=""

#-- Email Notifications Addresses --#
PIPELINE_NOTIFICATION_MAIL=""
PIPELINE_MANUAL_APPROVER_MAIL=""

#####################################

#-- AWS Configuration --#
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
AWS_REGION=""

#-- Docker Hub Auth (For Pull Limits) --#
DOCKERHUB_USERNAME=""
DOCKERHUB_PASSWORD=""
```

<br/>

## Fully Documented Deployment Walkthrough

This section provides a **comprehensive step-by-step deployment walkthrough** of the AWS DevSecOps Hybrid CI/CD Platform, demonstrating the complete deployment process from initial setup through final verification. The walkthrough includes real screenshots from a live deployment, showing the actual execution flow and expected outputs.

### Prerequisites Verification & Initial Setup

The deployment process begins with the automated deployment script validating all prerequisites and initializing the environment configuration.

<div align="center">

![Deployment Script Help](./doc/1_1-deployment_script_help.png)

*Figure 12: Deployment Script Help - Command-line options and usage instructions for the automated deployment script*

</div>

The deployment script provides comprehensive help documentation, showing all available options including full deployment, infrastructure skipping, and rollback capabilities.

<div align="center">

![Deployment Script Initialization](./doc/1_2-deployment_script_init.png)

*Figure 13: Deployment Script Initialization - Prerequisites validation and environment configuration loading*

</div>

The script performs thorough validation of all required tools (AWS CLI, Terraform, jq, zip utilities) and loads environment configuration from the `.env` file with comprehensive error checking.

### Terraform Infrastructure Deployment

Once prerequisites are validated, the deployment script initiates the Terraform infrastructure provisioning process.

<div align="center">

![Terraform Initialization](./doc/1_3-terraform_initialization.png)

*Figure 14: Terraform Initialization - Terraform backend initialization and provider plugin installation*

</div>

Terraform initializes the working directory, downloading required provider plugins and setting up the backend configuration for state management.

<div align="center">

![Terraform Platform Planning](./doc/1_4-terraform_platform_planning.png)

*Figure 15: Terraform Platform Planning - Infrastructure planning phase showing resources to be created*

</div>

The Terraform planning phase analyzes the configuration and displays all resources that will be created, including VPC components, ECS clusters, security groups, and supporting infrastructure.

<div align="center">

![Successful Platform Resources Deployment](./doc/1_5-successful_platform_resources_deployment.png)

*Figure 16: Successful Platform Resources Deployment - Terraform apply completion with created infrastructure summary*

</div>

Terraform successfully provisions the complete infrastructure foundation, creating all networking, compute, and storage resources as defined in the modular configuration.

<div align="center">

![Successful Terraform to CloudFormation Deployments](./doc/1_6-successful_terraform_to_cloudformation_deployments.png)

*Figure 17: Successful Terraform to CloudFormation Integration - Hybrid IaC deployment completion with output capture*

</div>

The deployment script successfully captures Terraform outputs and initiates the CloudFormation pipeline deployment, demonstrating the seamless integration between the two IaC tools.

<div align="center">

![Deployment Rollback Execution](./doc/1_7-deployment_rollback_execution.png)

*Figure 18: Deployment Rollback Execution - Complete infrastructure and pipeline rollback to previous stable state*

</div>

The rollback functionality demonstrates the automated cleanup capabilities, ensuring complete resource removal when testing or rolling back deployments.

### CloudFormation Stack Deployment & AWS Service Integration

Following successful infrastructure provisioning, the deployment process transitions to CloudFormation stack creation and AWS service orchestration.

<div align="center">

![CloudFormation Stack Completed](./doc/2-1_cloudformation_stack_completed.png)

*Figure 19: CloudFormation Stack Deployment Completion - Complete CI/CD pipeline stack creation with all AWS services*

</div>

The CloudFormation stack successfully creates all pipeline components including CodePipeline, CodeBuild projects, Lambda functions, SNS topics, and IAM roles with proper cross-service integration.

<div align="center">

![CloudFormation Stack Infrastructure Composer](./doc/2-2_cloudformation_stack_infrastructure_composer.png)

*Figure 20: CloudFormation Infrastructure Composer - Visual representation of deployed AWS resources and their dependencies*

</div>

The AWS CloudFormation Infrastructure Composer provides a comprehensive visual representation of all deployed resources, showing the complex interconnections between CodePipeline stages, security services, and monitoring components.

<div align="center">

![CloudFormation Resource Creation Graph](./doc/2-3_cloudformation_resource_creation_graph.png)

*Figure 21: CloudFormation Resource Creation Graph - Resource dependency mapping and creation sequence visualization*

</div>

The resource creation graph illustrates the sophisticated dependency management within the CloudFormation template, ensuring proper resource creation order and cross-service references.

### CodeConnections Setup & Repository Integration

Post-deployment configuration requires manual setup of the AWS CodeConnections connection for secure repository integration.

<div align="center">

![CodeConnections Connection Setup](./doc/3_codeconnections_connection_setup.png)

*Figure 22: CodeConnections Connection Setup - Manual authorization step for secure Git repository integration*

</div>

The CodeConnections setup requires one-time manual authorization through the AWS Console, establishing secure webhook-based integration between the CI/CD pipeline and the external Git repository.

### ECS Cluster Infrastructure Verification

The deployment verification includes confirmation of ECS cluster infrastructure and container orchestration capabilities.

<div align="center">

![ECS Clusters](./doc/5_ecs_clusters.png)

*Figure 23: ECS Clusters Overview - Staging and production clusters with EC2-backed container instances*

</div>

The ECS clusters demonstrate the separation between staging and production environments, with the staging cluster configured for ephemeral operation (scaling to zero when not in use) and the production cluster maintaining persistent capacity.

### S3 Storage Infrastructure & Artifact Management

The platform's storage infrastructure provides centralized artifact management and Lambda function packaging.

<div align="center">

![Platform S3 Buckets](./doc/6_1_platform_s3_buckets.png)

*Figure 24: Platform S3 Buckets - Artifact storage and Lambda function packaging infrastructure*

</div>

The S3 bucket architecture includes dedicated buckets for pipeline artifacts, Lambda function packages, and CloudTrail logging, all configured with appropriate encryption and lifecycle policies.

<div align="center">

![Artifact Store Bucket](./doc/6_2_artifact_store_bucket.png)

*Figure 25: Artifact Store Bucket Contents - Centralized storage for all pipeline artifacts and security scan results*

</div>

The artifact store bucket demonstrates the organized storage structure for pipeline artifacts, with automatic encryption and versioning enabled for audit trail maintenance.

<div align="center">

![Encrypted Artifacts Archive](./doc/6_3_encrypted_artifacts_archive.png)

*Figure 26: Encrypted Artifacts Archive - Security scan results and build artifacts with KMS encryption*

</div>

All artifacts are automatically encrypted using the dedicated KMS key, ensuring security compliance for stored security scan results, build artifacts, and deployment packages.

### CI/CD Pipeline Execution & Security Integration

The complete CI/CD workflow demonstrates end-to-end security scanning and automated deployment capabilities.

<div align="center">

![Successful CI/CD Pipeline Run](./doc/7_1_successful_CICD_pipeline_run.png)

*Figure 27: Successful CI/CD Pipeline Execution - Complete security scanning and deployment workflow*

</div>

The successful pipeline execution shows all stages completing successfully, from source checkout through security scanning, staging deployment, DAST analysis, manual approval, and production deployment.

<div align="center">

![Runner Example DAST Scans](./doc/7_2_runner_example-DAST_scans.png)

*Figure 28: DAST Security Scanning Results - OWASP ZAP penetration testing against staging environment*

</div>

The DAST scanning stage demonstrates comprehensive security testing against the live staging application, with OWASP ZAP performing automated penetration testing and vulnerability assessment.

<div align="center">

![Runner Example Production Blue-Green Deployment](./doc/7_3_runner_example-production_BLUE_GREEN_deployment.png)

*Figure 29: Production Blue-Green Deployment - Zero-downtime production release with automated traffic switching*

</div>

The blue-green deployment implementation ensures zero-downtime production releases through automated ECS service updates and Application Load Balancer target group management.

### Successful Production Deployment Verification

The deployment verification confirms successful application deployment and monitoring capabilities.

<div align="center">

![Successful Production Deployment](./doc/8_1_successful_production_deployment.png)

*Figure 30: Successful Production Deployment - Live application running in production environment*

</div>

The production deployment verification shows the application successfully running in the production ECS cluster with proper health checks and load balancer integration.

<div align="center">

![Successful Production Deployment 2](./doc/8_2_successful_production_deployment_2.png)

*Figure 31: Successful Production Deployment x2 - Live application demonstrating successful full API deployment and functionality*

</div>

The live application interface confirms successful deployment with all application features functioning correctly in the production environment.

### Security Hub Integration & Vulnerability Management

The platform's security integration provides centralized vulnerability management and compliance monitoring.

<div align="center">

![Security Hub Centralized Vulnerability Findings](./doc/9_1_security_hub_centralized_vulnerability_findings.png)

*Figure 32: Security Hub Centralized Findings - Normalized security scan results from all pipeline stages*

</div>

AWS Security Hub provides centralized visibility into all security findings from the multi-stage scanning process, with normalized vulnerability data from Snyk SAST, Clair SCA, and OWASP ZAP DAST tools.

<div align="center">

![Example SAST Finding](./doc/9_2_example_SAST_finding.png)

*Figure 33: SAST Security Finding Example - Detailed vulnerability analysis with remediation guidance*

</div>

Individual security findings provide detailed analysis including vulnerability classification, severity assessment, and specific remediation guidance for development teams.

### CloudWatch Monitoring & Observability

The platform's monitoring infrastructure provides comprehensive operational visibility and security monitoring.

<div align="center">

![CloudWatch RASP Falco Logs](./doc/10_1_cloudwatch_RASP_falco_logs.png)

*Figure 34: CloudWatch RASP Falco Logs - Runtime Application Security Protection monitoring*

</div>

CNCF Falco provides real-time runtime security monitoring through CloudWatch integration, detecting anomalous behavior and security threats during application execution.

<div align="center">

![CloudWatch Lambda Run Example](./doc/10_2_cloudwatch_lambda_run_example.png)

*Figure 35: CloudWatch Lambda Execution Logs - Security data processing and normalization workflow*

</div>

The Lambda function execution logs demonstrate the security data processing workflow, showing successful normalization and aggregation of security scan results.

<div align="center">

![CloudTrail Log Group](./doc/10_3_cloudtrail_logGroup.png)

*Figure 36: CloudTrail Log Group - Comprehensive audit logging for compliance and security monitoring*

</div>

CloudTrail integration provides comprehensive audit logging for all API activities, ensuring complete compliance tracking and security monitoring across the entire platform.

<div align="center">

![CloudWatch Pipeline Log Group Streams](./doc/10_4_cloudwatch_pipeline_logGroup_streams.png)

*Figure 37: CloudWatch Pipeline Log Streams - Dedicated logging for each pipeline stage and security analysis*

</div>

The dedicated log streams provide organized logging for each pipeline component, enabling detailed troubleshooting and operational monitoring across all DevSecOps stages.

### SNS Email Subscriptions & Notification System

The platform's notification system ensures stakeholder awareness and approval workflows.

<div align="center">

![SNS Email Subscriptions](./doc/4_sns_email_subscriptions.png)

*Figure 38: SNS Email Subscriptions - Multi-topic notification system for pipeline events and security alerts*

</div>

The SNS email subscription system provides automated notifications for pipeline state changes, security findings, manual approval requirements, and operational alerts, ensuring appropriate stakeholder engagement throughout the deployment process.

---

**`Deployment Success Verification:`**

The complete deployment walkthrough demonstrates:

- **Infrastructure Provisioning**: Successful Terraform deployment with proper resource creation and output capture
- **Pipeline Integration**: Seamless CloudFormation deployment with automated parameter injection from Terraform outputs
- **Security Implementation**: Comprehensive multi-stage security scanning with centralized vulnerability management
- **Monitoring & Compliance**: Complete observability stack with audit logging and runtime security monitoring
- **Operational Readiness**: Production-grade application deployment with zero-downtime release capabilities

The AWS DevSecOps Hybrid CI/CD Factory provides enterprise-grade software delivery with integrated security, compliance, and operational monitoring, ready for immediate production use.

---

<br/>

> © Project by [Haitam Bidiouane](https://linkedin.com/in/haitam-bidiouane/) - 2025

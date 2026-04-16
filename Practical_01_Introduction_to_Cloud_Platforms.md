# Practical 01 — Introduction to Cloud Platforms

---

## 📌 Objective

Familiarize yourself with the three major cloud platforms — **AWS**, **Azure**, and **GCP** — by creating free-tier accounts, exploring their dashboards, identifying key services, and understanding pricing calculators.

---

## 🧠 Conceptual Background (Know-How)

### What is Cloud Computing?

Cloud computing is the on-demand delivery of IT resources — compute power, storage, databases, networking, and more — over the internet with pay-as-you-go pricing. Instead of owning physical hardware, you rent virtual infrastructure from a cloud provider.

### The Three Major Providers

| Feature | AWS | Azure | GCP |
|---|---|---|---|
| Full Name | Amazon Web Services | Microsoft Azure | Google Cloud Platform |
| Founded | 2006 | 2010 | 2011 |
| Market Share | ~31% | ~25% | ~11% |
| Compute Service | EC2 | Virtual Machines | Compute Engine |
| Storage | S3 | Blob Storage | Cloud Storage |
| Managed Kubernetes | EKS | AKS | GKE |
| Serverless | Lambda | Azure Functions | Cloud Functions |

### Cloud Service Models

```
┌─────────────────────────────────────────────┐
│              SaaS (Software)                │  ← Gmail, Office 365
├─────────────────────────────────────────────┤
│              PaaS (Platform)                │  ← App Engine, Elastic Beanstalk
├─────────────────────────────────────────────┤
│         IaaS (Infrastructure)               │  ← EC2, VMs, Compute Engine
└─────────────────────────────────────────────┘
       You manage less as you go higher up
```

### Cloud Deployment Models

- **Public Cloud** — Shared infrastructure managed by the provider (AWS, Azure, GCP)
- **Private Cloud** — Dedicated infrastructure for one organization (OpenStack)
- **Hybrid Cloud** — Combination of public and private
- **Multi-Cloud** — Using services from multiple providers simultaneously

---

## 🛠️ Step-by-Step Guide

### Step A — Create Free-Tier Accounts

#### AWS Free Tier
1. Go to [https://aws.amazon.com/free](https://aws.amazon.com/free)
2. Click **Create a Free Account**
3. Enter your email and choose an account name
4. Provide payment details (needed for identity verification, not charged for free tier)
5. Complete phone verification
6. Select the **Basic Support** plan (free)
7. Sign in to the **AWS Management Console**

> ⚠️ **Free Tier Limits (AWS):**
> - 750 hours/month of EC2 t2.micro (12 months)
> - 5 GB of S3 standard storage
> - 1 million Lambda requests/month (always free)

#### Azure Free Tier
1. Go to [https://azure.microsoft.com/free](https://azure.microsoft.com/free)
2. Click **Start free**
3. Sign in with a Microsoft account or create one
4. Provide identity and payment verification
5. You receive **$200 credit** for 30 days + always-free services

> ⚠️ **Free Tier Limits (Azure):**
> - $200 credit for first 30 days
> - 750 hours of B1S Linux VMs (12 months)
> - 5 GB Blob storage (12 months)

#### GCP Free Tier
1. Go to [https://cloud.google.com/free](https://cloud.google.com/free)
2. Click **Get started for free**
3. Sign in with a Google account
4. Provide billing info
5. You receive **$300 credit** for 90 days

> ⚠️ **Free Tier Limits (GCP):**
> - $300 credit for 90 days
> - 1 e2-micro VM instance (always free, us-central1 region)
> - 5 GB Cloud Storage (always free)

---

### Step B — Explore Dashboards and Key Services

#### AWS Management Console

**Navigation:**
```
Console Home → Services Menu (top-left) → Search or browse categories
```

**Key Services to Explore:**

| Category | Service Name | What It Does |
|---|---|---|
| Compute | EC2 | Virtual machines |
| Compute | Lambda | Serverless functions |
| Storage | S3 | Object/file storage |
| Storage | EBS | Block storage (disks for EC2) |
| Networking | VPC | Virtual private network |
| Networking | Route 53 | DNS service |
| Database | RDS | Managed relational DB |
| Database | DynamoDB | Managed NoSQL DB |
| Security | IAM | Identity & access management |
| Monitoring | CloudWatch | Metrics, logs, alarms |

**Hands-on Exploration Checklist:**
- [ ] Navigate to EC2 → view the instance types available
- [ ] Navigate to S3 → see how buckets are organized
- [ ] Navigate to IAM → explore users, roles, policies
- [ ] Navigate to VPC → view default VPC setup
- [ ] View the billing dashboard under your account name

#### Azure Portal

**Navigation:**
```
portal.azure.com → Top search bar or left sidebar → All Services
```

**Key Services to Explore:**

| Category | Service Name | AWS Equivalent |
|---|---|---|
| Compute | Virtual Machines | EC2 |
| Compute | Azure Functions | Lambda |
| Storage | Blob Storage | S3 |
| Networking | Virtual Network | VPC |
| Database | Azure SQL | RDS |
| Security | Azure AD | IAM |
| Monitoring | Azure Monitor | CloudWatch |

#### GCP Console

**Navigation:**
```
console.cloud.google.com → Hamburger menu (top-left) → Browse services
```

**Key Services to Explore:**

| Category | Service Name | AWS Equivalent |
|---|---|---|
| Compute | Compute Engine | EC2 |
| Compute | Cloud Functions | Lambda |
| Storage | Cloud Storage | S3 |
| Networking | VPC Network | VPC |
| Database | Cloud SQL | RDS |
| Database | Firestore | DynamoDB |
| Monitoring | Cloud Monitoring | CloudWatch |

---

### Step C — Understand Pricing Calculators

#### AWS Pricing Calculator
- **URL:** [https://calculator.aws/pricing/2/home](https://calculator.aws/pricing/2/home)
- Click **Create estimate**
- Add a service (e.g., EC2)
- Configure: region, instance type, hours/month, OS
- Add S3, data transfer to see a full estimate

**Example EC2 Estimate (t2.micro, us-east-1, Linux, 730 hrs/month):**
```
EC2 Instance:     ~$8.47/month
EBS (8 GB gp2):   ~$0.80/month
Data Transfer:    ~$0.00 (first 100 GB free outbound)
─────────────────────────────
Total:            ~$9.27/month
```

#### Azure Pricing Calculator
- **URL:** [https://azure.microsoft.com/pricing/calculator](https://azure.microsoft.com/pricing/calculator)
- Select a product → Configure region, tier, hours
- Compare Dev/Test pricing vs. production pricing

#### GCP Pricing Calculator
- **URL:** [https://cloud.google.com/products/calculator](https://cloud.google.com/products/calculator)
- Add product → fill configuration
- Supports committed use discounts and sustained use discounts automatically

**Cost Optimization Concepts:**

| Strategy | Description |
|---|---|
| On-Demand | Pay per hour, no commitment |
| Reserved Instances | 1–3 year commitment, up to 72% savings (AWS) |
| Spot/Preemptible | Use spare capacity, up to 90% savings, can be interrupted |
| Savings Plans | Flexible commitment on compute spend (AWS) |

---

## 📊 Comparison Summary

```
          AWS              AZURE              GCP
        ┌──────┐         ┌──────┐          ┌──────┐
Free    │$0    │         │$200  │          │$300  │
Credit  │      │         │30days│          │90days│
        └──────┘         └──────┘          └──────┘

Strengths:
AWS   → Widest service catalog, most mature, largest community
Azure → Best for Microsoft/Windows shops, Active Directory integration
GCP   → Best pricing for compute, strongest in ML/AI (BigQuery, Vertex AI)
```

---

## ✅ Learning Outcomes

After completing this practical, you should be able to:

- [ ] Create accounts on all three major cloud platforms
- [ ] Navigate the console/portal of each platform confidently
- [ ] Identify the equivalent service across platforms (e.g., EC2 = VM = Compute Engine)
- [ ] Use pricing calculators to estimate monthly costs
- [ ] Understand the difference between IaaS, PaaS, and SaaS
- [ ] Explain on-demand vs. reserved vs. spot pricing models

---

## 🔐 Security Best Practices (Day 1 Habits)

1. **Never use the root account** for daily tasks on AWS — create an IAM user immediately
2. **Enable MFA** on all accounts
3. **Never share credentials** or commit them to GitHub
4. **Set billing alerts** to avoid unexpected charges (AWS: CloudWatch billing alarm; Azure: Cost alerts; GCP: Budget alerts)

---

## 📚 Further Reading

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)
- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
- [Cloud Computing Concepts — NIST Definition](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-145.pdf)

# Terraform Is Cloud-Agnostic: Multi-Cloud Guide (AWS + Azure + GCP)

> **TL;DR**: Terraform is **provider-based** and **cloud-agnostic**. You can safely define and deploy resources across AWS, Azure, and GCP **in a single configuration**, while keeping your workflow consistent with HCL, remote state, and workspaces.

---

## ✅ What “Cloud-Agnostic” Means in Terraform

- **Providers** abstract interactions with platforms (AWS, Azure, GCP, Kubernetes, VMware, GitHub, etc.).
- You can **mix multiple providers** in one configuration and **compose dependencies** between them.
- The **same CLI workflow** (`init`, `plan`, `apply`) works across providers.

**Common caveats**:
- Each provider has **cloud-specific resources/quirks** (e.g., IAM vs. RBAC).
- Credentials and network access must be handled per cloud.
- Cross-cloud dependencies should be explicit (outputs/variables or data sources).

---

## 🧰 Prerequisites

- **Terraform** v1.5+ installed.
- Accounts/credentials for **AWS**, **Azure**, **GCP**:
  - **AWS**: `AWS_PROFILE` *or* `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_DEFAULT_REGION`
  - **Azure**: `ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID`
  - **GCP**: `GOOGLE_APPLICATION_CREDENTIALS` (path to a JSON service account key), `GOOGLE_PROJECT`, `GOOGLE_REGION`, `GOOGLE_ZONE`
- Optional: **Terraform Cloud / HCP Terraform** account (for remote state and workspaces).
  - Run `terraform login` (details below).

---

## 📁 Project Structure

```
multi-cloud/
├─ main.tf
├─ versions.tf
├─ providers.tf
├─ variables.tf
├─ outputs.tf
└─ modules/
   ├─ aws_ec2/
   │  ├─ main.tf
   │  ├─ variables.tf
   │  └─ outputs.tf
   ├─ azure_vm/
   │  ├─ main.tf
   │  ├─ variables.tf
   │  └─ outputs.tf
   └─ gcp_instance/
      ├─ main.tf
      ├─ variables.tf
      └─ outputs.tf
```

---
##  One-Line Difference between `required_providers` and `provider`
required_providers → declares dependency : `required_providers` defines WHAT provider and version is required
provider           → configures connection :  `provider` defines HOW Terraform connects to that provider

## 🔧 Root Configuration (HCL)

### `versions.tf`
```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }

  cloud {
    organization = "your-org"
    workspaces {
      name = "multi-cloud-demo"
    }
  }
}
```

### `providers.tf`
```hcl
provider "aws" {
  region = var.aws_region
}

provider "azurerm" {
  features {}
  subscription_id = var.azure_subscription_id
}

provider "google" {
  project = var.gcp_project
  region  = var.gcp_region
  zone    = var.gcp_zone
}
```

### `variables.tf`
```hcl
variable "tags" {
  type        = map(string)
  default     = { owner = "platform", purpose = "demo" }
}

variable "aws_region" { default = "us-east-1" }
variable "azure_subscription_id" {}
variable "gcp_project" {}
variable "gcp_region" { default = "us-central1" }
variable "gcp_zone" { default = "us-central1-a" }
```

### `main.tf`
```hcl
module "aws_ec2" {
  source     = "./modules/aws_ec2"
  aws_region = var.aws_region
  tags       = var.tags
}

module "azure_vm" {
  source                = "./modules/azure_vm"
  azure_subscription_id = var.azure_subscription_id
  tags                  = var.tags
}

module "gcp_instance" {
  source      = "./modules/gcp_instance"
  gcp_project = var.gcp_project
  gcp_region  = var.gcp_region
  gcp_zone    = var.gcp_zone
  labels      = var.tags
}
```

---

## 🧩 Module Examples

### AWS EC2 Module
```hcl
resource "aws_instance" "vm" {
  ami           = "ami-123456"
  instance_type = "t3.micro"
}
```

### Azure VM Module
```hcl
resource "azurerm_linux_virtual_machine" "vm" {
  name     = "tf-azure-vm"
  size     = "Standard_B1s"
}
```

### GCP Instance Module
```hcl
resource "google_compute_instance" "vm" {
  name         = "tf-gcp-vm"
  machine_type = "e2-micro"
}
```

---

## ☁️ Remote State with Terraform Cloud

Run:
```bash
terraform login
```

Configure `cloud {}` block in `versions.tf`.

---

## 🧪 Workspaces
```bash
terraform workspace new dev
terraform workspace select dev
```

---

## ⚠️ Multi-Cloud Gotchas
- Credentials isolation
- Provider aliases for multi-region
- Cross-cloud dependencies via outputs

---

## 💡 Tips & Tricks
- Use `tflint`, `tfsec`
- Remote state for collaboration
- Destroy after demo to save cost

---

## 🚀 Runbook
```bash
terraform init
terraform plan
terraform apply
terraform destroy
```

---

## 🧠 MCQs

1. Terraform is cloud-agnostic because of:
   - A) AWS-only support
   - B) Separate HCL per cloud
   - C) **Provider abstraction layer** ✅

2. Can AWS and Azure resources coexist in one file?
   - A) **True** ✅

3. `terraform login` does:
   - A) **Stores Terraform Cloud token** ✅

...

---

## 📌 Key Takeaways
- Terraform supports multi-cloud in one config.
- Use remote state and workspaces.
- Handle credentials and dependencies carefully.

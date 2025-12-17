# HCP Sentinel Comprehensive Guide

## ✅ What is HCP Sentinel?
HCP Sentinel is a **Policy-as-Code (PaC)** framework integrated into HashiCorp products (Terraform, Vault, Consul, Nomad) and available as a managed service in HCP. It allows organizations to enforce **governance, compliance, and security policies** automatically during infrastructure provisioning and operations.

---

## 🔍 What Can It Do?
- **Policy Enforcement**: Define rules for infrastructure resources (e.g., restrict instance sizes, enforce tagging).
- **Compliance Checks**: Ensure deployments meet organizational or regulatory standards.
- **Cost Control**: Prevent provisioning of expensive resources.
- **Security Controls**: Enforce encryption, network restrictions, etc.
- **Integration**: Works with Terraform Cloud/Enterprise, Vault, Consul, Nomad.

---

## ⚙️ How Does It Work?
1. **Policy Definition**:
   - Written in **Sentinel language** (similar to HCL but for policies).
   - Example: Restrict AWS instance type to `t2.micro`.

2. **Policy Sets**:
   - Group multiple policies into a set.
   - Attach to **Terraform workspace** or **Vault namespace**.

3. **Enforcement Levels**:
   - **Advisory**: Warn but allow.
   - **Soft Mandatory**: Block unless overridden.
   - **Hard Mandatory**: Block and cannot override.

4. **Execution Flow**:
   - When you run `terraform plan` or `apply`, Sentinel evaluates policies **before changes are applied**.

---

## 🛠 How to Use HCP Sentinel
### Step 1: Enable Sentinel in HCP
- Go to **HCP Console → Sentinel**.
- Create a **Policy Set**.

### Step 2: Write a Policy
Example: Restrict AWS instance type:
```hcl
import "tfplan/v2" as tfplan

main = rule {
    all tfplan.resources.aws_instance as _, instances {
        all instances as _, instance {
            instance.applied.type is "t2.micro"
        }
    }
}
```

### Step 3: Attach Policy Set
- Attach to Terraform workspace in HCP/Terraform Cloud.

### Step 4: Test Policy
- Use `sentinel test` locally or via Terraform Cloud.

---

## ✅ Benefits
- Centralized governance.
- Automated compliance.
- Reduces human error.
- Scales across teams.

---

## 💡 Tips & Tricks
- Use **mock data** for testing policies without running full Terraform.
- Combine **multiple imports** like `tfplan`, `tfconfig`, `tfstate` for comprehensive checks.
- Start with **Advisory** mode for gradual adoption.

---

## 🔐 Advanced Examples
### 1. Cost Control Policy
Prevent provisioning of expensive EC2 instances:
```hcl
import "tfplan/v2" as tfplan

main = rule {
    all tfplan.resources.aws_instance as _, instances {
        all instances as _, instance {
            instance.applied.type in ["t2.micro", "t2.small"]
        }
    }
}
```

### 2. Security Policy
Ensure all S3 buckets have encryption enabled:
```hcl
import "tfplan/v2" as tfplan

main = rule {
    all tfplan.resources.aws_s3_bucket as _, buckets {
        all buckets as _, bucket {
            bucket.applied.server_side_encryption_configuration is not null
        }
    }
}
```

---

## 📚 MCQs for Practice
1. **What is Sentinel primarily used for?**
   - A) Monitoring infrastructure
   - B) Policy-as-Code enforcement
   - C) Logging Terraform runs
   - D) Encrypting secrets  
   **Answer:** B

2. **Which language is used to write Sentinel policies?**
   - A) Python
   - B) HCL
   - C) Sentinel language
   - D) YAML  
   **Answer:** C

3. **Which enforcement level cannot be overridden?**
   - A) Advisory
   - B) Soft Mandatory
   - C) Hard Mandatory
   - D) Optional  
   **Answer:** C

4. **Sentinel integrates with which HashiCorp products?**
   - A) Terraform only
   - B) Vault, Consul, Nomad, Terraform
   - C) Kubernetes only
   - D) None  
   **Answer:** B

5. **What command is used to test Sentinel policies locally?**
   - A) `terraform test`
   - B) `sentinel test`
   - C) `hcp sentinel`
   - D) `policy check`  
   **Answer:** B

---

# 🔐 Terraform: `tls_private_key` vs `ssh-keygen` for EC2 SSH Access

When provisioning EC2 instances with Terraform, you must decide **how to generate and manage SSH keys**.  

#### 📌 Full working examples with EC2 :  
> 👉 https://github.com/mohakhan03uk/terraform/blob/main/aws/ManualKeyPairAndItsUseInEC2.md   \
> 👉 https://github.com/mohakhan03uk/terraform/blob/main/aws/tls_private_key_AndItsUseInEC2.md 

This guide compares two common approaches:

---

## 🔑 1. Using `ssh-keygen` (Manual Local Key Generation)

You generate your keys **locally**:

```bash
ssh-keygen -t rsa -b 4096 -f mykey
```

Then upload the **public key** to AWS via Terraform:

```hcl
resource "aws_key_pair" "uploaded" {
  key_name   = "my-uploaded-key"
  public_key = file("mykey.pub")
}
```

### ✔ Pros
- Private key stays safely on your machine  
- No sensitive material in Terraform state  
- Strong for compliance‑heavy environments  

### ✖ Cons
- Requires manual step  
- Not ideal for high automation or many instances  

---

## 🔐 2. Using `tls_private_key` (Terraform‑Generated Keys)

Terraform generates key pairs for you inside your IaC workflow:

```hcl
resource "tls_private_key" "generated" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "auto" {
  key_name   = "tf-generated-key"
  public_key = tls_private_key.generated.public_key_openssh
}
```
### Save Private Key to Disk (secure permissions)
```hcl
resource "local_file" "private_key" {
  content         = tls_private_key.generated.private_key_pem
  filename        = pathexpand("~/.ssh/mykey_tls")
  file_permission = "0600"
}
```

### ✔ Pros
- Fully automated  
- Consistent and repeatable for large deployments  
- Great for CI/CD pipelines  

### ✖ Cons
- Private key is stored in Terraform state  
- Requires secure remote state management  
- Must handle rotations carefully  

---

## 🧠 Summary Table

| Method | Automation | Key Safety | Best Use Case |
|--------|------------|-------------|----------------|
| **`ssh-keygen`** | Manual | ⭐⭐⭐⭐⭐ | High‑security or regulated workloads |
| **`tls_private_key`** | Fully automated | ⭐⭐⭐ | Scalable IaC pipelines |

---

## 🏁 Recommendation

- Choose **`ssh-keygen`** when key protection is the priority.  
- Choose **`tls_private_key`** when automation, scaling, and reproducibility matter most.

---

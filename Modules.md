
## 1. Introduction to Terraform Modules

Terraform Modules allow us to **centralize resource configuration** and make it easier for multiple projects to **reuse Terraform code**.

Instead of writing infrastructure code from scratch every time, we can use:
- Custom-built internal modules
- Ready-made modules from the Terraform Registry

### Example Concept
```
VPC Module + EC2 Module + ALB Module
        ↓
   terraform apply
        ↓
 Complete infrastructure
```

---

## 2. Root Module vs Child Module

### Root Module
- Resides in the **main working directory**
- Entry point for Terraform commands (`init`, `plan`, `apply`)

### Child Module
- A module **called by another module**
- Encapsulates reusable logic

```hcl
module "ecRoot" {
  source = "../../modules/ec2Child"
}
```

---

## 3. Calling a Module

Modules are referenced using the **module block**.

### Mandatory Argument
- `source` – location of the module

```hcl
module "ec2" {
  source = "SOURCE_LOCATION"
}
```

---

## 4. Module Source Code Locations

Module source code can be present in a wide variety of locations.

### 4.1 Local Path Module
A local path must begin with `./` or `../`.

```hcl
module "ec2" {
  source = "../modules/ec2"
}
```

---

### 4.2 Generic Git Repository
Arbitrary Git repositories can be used with the `git::` prefix.

```hcl
module "vpc" {
  source = "git::https://example.com/vpc.git"
}
```

---

### 4.3 Terraform Registry
Recommended and most common approach.

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.11.1"
}
```

✔ Always pin module versions.

---

### 4.4 HTTP URLs

```hcl
module "vpc" {
  source = "https://moduleFQDN.com/vpc-module.zip"
}
```

---

### 4.5 S3 Buckets

```hcl
module "vpc" {
  source = "s3::https://s3.amazonaws.com/my-bucket/vpc.zip"
}
```

---

## 5. Module Inputs (Variables)

Some modules require **specific inputs** from the user.

### variables.tf
```hcl
variable "ami" {
  type = string
}

variable "instance_type" {
  type = string
}
```

### main.tf
```hcl
resource "aws_instance" "ec2222" {
  ami           = var.ami
  instance_type = var.instance_type
}
```

---

## 6. Best Practices

- ❌ Do not use hard-coded values
- ✅ Always use variables
- ❌ Avoid hard-coding region inside modules
- ✅ Define provider version constraints
- ✅ Make modules reusable and generic

---

## 7. Required Providers in Modules

Every module should declare required providers.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.8"
    }
  }
}
```

📌 Provider **configuration** belongs in the **root module**, not child modules.

---

## 8. Output Values

Output values:
- Display information after `terraform apply`
- Expose data to other modules

### Example
```hcl
resource "aws_eip" "lb" {
  domain = "vpc"
}

output "public_ip" {
  value = aws_eip.lb.public_ip
}
```

---

## 9. Accessing Child Module Outputs

Format:
```text
module.<MODULE_NAME>.<OUTPUT_NAME>
```

### Example
```hcl
module "ec2" {
  source = "../../modules/ec2"
}

resource "aws_eip" "lb" {
  instance = module.ec2.instance_id
  domain   = "vpc"
}
```

---

## 10. Module Structure

### Minimal Module Structure
```
minimal-module/
├── README.md
├── main.tf
├── variables.tf
└── outputs.tf
```

---

### Standard Module Structure
```
complete-module/
├── README.md
├── main.tf
├── variables.tf
├── outputs.tf
├── modules/
│   ├── nestedA/
│   └── nestedB/
├── examples/
│   ├── exampleA/
│   └── exampleB/
```

---

## 11. Publishing Modules to Terraform Registry

### Requirements
1. Public GitHub repository :  The module must be on GitHub and must be a public repo. This is only a requirement for the public registry.
2. Naming convention:
   ```
   terraform-<PROVIDER>-<NAME>
   ```
   NAME can have string xyz-abc
3. Standard module structure
4. README.md with inputs/outputs/examples
5. Semantic version tags:  which can optionally be prefixed with a v. For example, v1.0.4 and 0.9.2
   ```
   v1.0.0
   v1.2.3
   ```

---

## 12. Module Composition (Modules Calling Modules)

Modules can call **other modules internally**.

```hcl
module "network" {
  source = "../vpc"
}

module "compute" {
  source    = "../ec2"
  subnet_id = module.network.public_subnet_id
}
```

---

## 13. Providers & Modules (Inheritance and Aliases)

- Child modules inherit providers from root
- Providers can be explicitly passed

```hcl
module "ec2" {
  source = "../modules/ec2"
  providers = {
    aws = aws.us_east
  }
}
```

---

## 14. count & for_each with Modules

```hcl
module "ec2" {
  source = "../modules/ec2"
  count  = 2
}
```

```hcl
module "ec2" {
  source   = "../modules/ec2"
  for_each = var.instances
}
```

---

## 15. Module Caching & terraform init

- Modules are downloaded to:
```
.terraform/modules/
```

- Run `terraform init` when:
  - Adding a new module
  - Changing module source
  - Updating module version

---

## 16. Real Example: VPC → EC2 → ALB Stack

### VPC Module
```hcl
output "vpc_id" {
  value = aws_vpc.this.id
}
```

### EC2 Module
```hcl
output "instance_id" {
  value = aws_instance.this.id
}
```

### Root Module Wiring
```hcl
module "vpc" {
  source = "./modules/vpc"
}

module "ec2" {
  source    = "./modules/ec2"
  subnet_id = module.vpc.public_subnet_id
}

module "alb" {
  source  = "./modules/alb"
  subnets = module.vpc.public_subnets
}
```

---

## 17. Multi‑Region Module Usage with Provider Aliases
### Alias Meta-Argument : Each provider can have one default configuration, and any number of alternate configurations that include an extra name segment (or "alias").
```hcl
provider "aws" {
  region = "us-west-1"
}

provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}
```

```hcl
module "vpc_us" {
  source    = "./modules/vpc"
  providers = { aws = aws.us_east }
}

module "vpc_eu" {
  source    = "./modules/vpc"
  providers = { aws = aws.eu_west }
}
```

---

## Module Data Flow – Advanced Understanding (Exam Critical)
---
Terraform modules follow **strict one-way data flow** rules.  
Child modules cannot access parent outputs directly. Data must always be passed explicitly via input variables.

### Allowed Data Flow
| Direction | Allowed | Mechanism |
|--------|--------|----------|
Parent → Child | ✅ | Input variables |
Child → Parent | ✅ | Output values |
Child → Parent outputs | ❌ | Not supported |

---

### ❌ Invalid (Not Allowed)

A child module **cannot reference** its parent or parent outputs.

```hcl
# ❌ NOT VALID – inside child module
resource "aws_instance" "example" {
  subnet_id = module.parent.vpc_id
}

## 18. Exam‑Focused MCQs (Modules Only)

### Q1
Which argument is mandatory in a module block?
A. version  
B. source  
C. providers  
D. depends_on  

**Answer:** B

### Q2
Where should provider configuration be defined?
A. Child module  
B. Root module  
C. outputs.tf  
D. README.md  

**Answer:** B

### Q3
How do you access a child module output?
A. var.module.output  
B. output.module.output  
C. module.name.output  
D. terraform.module.output  

**Answer:** C

### Q4
Which command downloads module source code?
A. terraform plan  
B. terraform apply  
C. terraform init  
D. terraform refresh  

**Answer:** C

### Q5
Modules are cached in which directory?
A. modules/  
B. .terraform/providers  
C. .terraform/modules  
D. .terraform/state  

**Answer:** C

---

## 19. Final Takeaways

- Root module = execution entry point
- Child modules = reusable building blocks
- Never hard-code values
- Providers configured in root
- Outputs enable module chaining
- Always pin module versions
- terraform init is mandatory for module changes

---

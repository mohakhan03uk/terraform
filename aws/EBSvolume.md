# Amazon EBS
Amazon Elastic Block Store (Amazon EBS) provides scalable, high-performance block storage resources that can be used with Amazon Elastic Compute Cloud (Amazon EC2) instances. With Amazon Elastic Block Store, you can create and manage the following block storage resources:

**Amazon EBS volumes** — These are storage volumes that you attach to Amazon EC2 instances. After you attach a volume to an instance, you can use it in the same way you would use a local hard drive attached to a computer, for example to store files or to install applications.

**Amazon EBS snapshots** — These are point-in-time backups of Amazon EBS volumes that persist independently from the volume itself. You can create snapshots to back up the data on your Amazon EBS volumes. You can then restore new volumes from those snapshots at any time.
## 📌 Are Amazon EBS volumes local to an Availability Zone?

**Yes.**  
Amazon Elastic Block Store (EBS) volumes are **Availability Zone (AZ)–local resources**.

An EBS volume:
- Is **created in exactly one Availability Zone**
- Can be **attached only to EC2 instances in the same AZ**
- Is **automatically replicated within that AZ** for durability

---

## 🧠 Core Characteristics

### Availability Zone Scope
- EBS volumes exist in **one and only one AZ**
- Cross-AZ attachment is **not supported**
- Enforced by AWS at the API level

### Durability Model
- EBS data is **replicated across multiple physical servers within the same AZ**
- Protects against:
  - Disk failure
  - Host failure
- Does **not** protect against:
  - AZ-wide failure

---


## 🔄 EBS Volume Lifecycle

### 1. Creation
- Created in a specific AZ
- Size, type, and encryption are defined at creation time
- Can be created:
  - Empty
  - From a snapshot
  - From an AMI (root volume)
#### Creating an [EBS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ebs_volume) using Terraform

```
Create an EBS Volume (AZ-local)
resource "aws_ebs_volume" "data_volume" {
  availability_zone = "us-east-1a"
  size              = 50
  type              = "gp3"
  encrypted         = true

  tags = {
    Name = "terraform-ebs"
  }
}
```
---
### 2. Attachment
- Can be attached to:
  - One EC2 instance (most volume types)
  - Multiple EC2 instances (io1/io2 with **Multi-Attach**)
- Instance **must be in the same AZ**
#### [Attaching EBS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/volume_attachment) to a EC2 intance 
```
resource "aws_volume_attachment" "ebs_attach" {
  device_name = "/dev/xvdf"
  volume_id   = aws_ebs_volume.data_volume.id
  instance_id = aws_instance.web.id
}
````
> ```aws ec2 attach-volume   --volume-id vol-0123456789abcdef0   --instance-id i-0123456789abcdef0 --device /dev/xvdf```
---
### 3. In-Use State
- Volume is mounted and used by the OS
- Supports:
  - Online resizing (most cases)
  - Performance modification (type, IOPS, throughput)
##### You can see the details of Volumes using below cmd :
> ```aws ec2 describe-volumes --region us-east-1```\
> ```aws ec2 describe-volumes --volume-ids   vol-0909125d472de2d77```

### 4. Detachment
- Can be detached:
  - Gracefully (recommended)
  - Force-detached (risk of data corruption)
- Data persists after detachment
- AWS :
> aws ec2 detach-volume  --volume-id vol-0123456789abcdef0 
- Terraform :
> EBS volumes are detached in Terraform by destroying the aws_volume_attachment resource; the volume itself remains intact unless explicitly deleted \
> *In production  Unmount filesystem first from EC2   by login into EC2 and umount*
---

### 5. Snapshot
- Point-in-time backup stored in S3 (managed by AWS)
- Incremental by nature
- Snapshots are **regional**, not AZ-bound
#### Snapshot Lifecycle with Terraform
```
resource "aws_ebs_snapshot" "backup" {
  volume_id = aws_ebs_volume.data_volume.id

  tags = {
    Name = "create-backup"
  }
}

# Snapshot resource is replaced on every Apply due to new value of timestamp()
Example:

resource "aws_ebs_snapshot" "backup_everyTime" {
  volume_id  = aws_ebs_volume.data_volume.id
  description = "backup-${timestamp()}"
}


```
> *backup* : will happen only on first terraform apply , if you want to take new one destroy and apply. \
> *backup_everyTime* : will happen every time when you run apply sure to timestamp()    or you can use DLM policy\
> ```aws ec2 create-snapshot  --volume-id vol-0123456789abcdef0 --description "Pre-maintenance snapshot" ```
---
### 6. Deletion
- Volume deletion is **permanent**
- Snapshots remain intact after volume deletion
- Root volumes may be auto-deleted depending on EC2 settings
- AWS :
  > Make sure the volume is detached : aws ec2 describe-volumes --volume-ids vol-0123456789abcdef0    : "State": "available" \
  > Delete the volume : aws ec2 delete-volume --volume-id vol-0123456789abcdef0
- Terraform :
  > Make sure the volume is detached :  Remove or destroy aws_volume_attachment first   : terraform destroy -target=aws_volume_attachment.data_attach \
  > Delete EBS volume resource : terraform destroy -target=aws_ebs_volume.data\
  > Then delete the EBS volume and attachement block from resource file
### 7. Restoring
#### Restore Snapshot into Another AZ
```
resource "aws_ebs_volume" "restored_volume" {
  availability_zone = "us-east-1b"
  snapshot_id       = aws_ebs_snapshot.backup.id
  type              = "gp3"
}
```
> aws ec2 create-volume   --snapshot-id snap-0123456789abcdef0  --availability-zone us-east-1b --volume-type gp3
---


## 🚫 Constraints and Limitations

### AZ Constraints
- Volume and EC2 instance must be in the **same AZ**
- No live migration across AZs

### Attachment Limits
- One volume → one EC2 instance (default)
- Multi-Attach supported only for:
  - `io1` and `io2`
  - Nitro-based instances
  - Linux only
  - Requires cluster-aware file systems

### Size Limits
- Minimum size: **1 GiB**
- Maximum size: **16 TiB** per volume

### Performance Constraints
- Performance depends on:
  - Volume type
  - Size
  - Provisioned IOPS / throughput
- Smaller volumes may not achieve max advertised performance

### OS-Level Constraints
- File system must be resized after volume expansion
- Force detach can lead to:
  - File system corruption
  - Data loss

---

## 🔐 Encryption

- EBS supports **encryption at rest, in transit, and at snapshot level**
- Uses **AWS KMS**
- Encryption is:
  - Transparent to applications
  - Mandatory in some organizations via SCPs
- Snapshots inherit encryption status

---

## 🔄 Cross-AZ and Cross-Region Usage

### How to Move an EBS Volume Across AZs
1. Create a snapshot
2. Restore snapshot in target AZ
3. Attach restored volume to EC2 in that AZ

### Cross-Region DR
- Copy snapshots to another region
- Restore during disaster recovery

---

## ⚠️ Failure Scenarios and Behavior

| Failure | EBS Behavior |
|------|-------------|
| Disk failure | Handled automatically |
| EC2 instance failure | Volume remains intact |
| AZ outage | Volume unavailable |
| Accidental deletion | Recoverable only via snapshot |

---

## 💰 Pricing and Billing Notes

- Charged for:
  - Provisioned storage (GB/month)
  - Provisioned IOPS (io1/io2)
  - Snapshots (GB/month)
- You are charged **even if the volume is unattached**
- Snapshots are incremental but billed for total unique data stored

---

## 🏗️ Best Practices

- Always:
  - Take regular snapshots
  - Enable encryption by default
- For high availability:
  - Use EBS + snapshots + automation
  - Or prefer **EFS / S3** for multi-AZ access
- Before detaching:
  - Unmount the file system
  - Flush buffers
- Tag volumes for cost tracking

---

## 🧪 Example Constraint Scenario

- EC2 instance: `us-east-1a`
- EBS volume: `us-east-1b`

❌ Attachment fails

Correct approach:
1. Snapshot volume in `us-east-1b`
2. Restore snapshot in `us-east-1a`
3. Attach restored volume

---

## 📚 Official AWS Documentation (Sources)

- EBS Volumes Overview  
  https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes.html

- Creating an EBS Volume  
  https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html

- EBS Features and Durability  
  https://docs.aws.amazon.com/ebs/latest/userguide/EBSFeatures.html

- EBS Snapshots  
  https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html

- Amazon EBS Product Page  
  https://aws.amazon.com/ebs/

---

## ✅ Summary (Exam / Interview Ready)

> **Amazon EBS volumes are AZ-local block storage resources that can only be attached to EC2 instances in the same Availability Zone, are replicated within that AZ for durability, and rely on snapshots for cross-AZ or cross-Region recovery.**

---















# 1
bob@iac-server ~/terraform via 💠 default ➜  cat main.tf  \

resource "aws_ebs_volume" "xfusion-volume" {\
\
}



bob@iac-server ~/terraform via 💠 default ✖ terraform plan \
╷\
│ Error: Missing required argument\
│ \
│   on main.tf line 1, in resource "aws_ebs_volume" "xfusion-volume":\
│    1: resource "aws_ebs_volume" "xfusion-volume" {\
│ \
│ The argument "availability_zone" is required, but no definition was found.


# 2
bob@iac-server ~/terraform via 💠 default ➜  cat main.tf 
resource "aws_ebs_volume" "xfusion-volume" {
 availability_zone = "us-east-1"

}

bob@iac-server ~/terraform via 💠 default ➜  terraform plan
╷
│ Error: Missing required argument
│ 
│   with aws_ebs_volume.xfusion-volume,
│   on main.tf line 1, in resource "aws_ebs_volume" "xfusion-volume":
│    1: resource "aws_ebs_volume" "xfusion-volume" {
│ 
│ "size": one of `size,snapshot_id` must be specified
╵
╷
│ Error: Missing required argument
│ 
│   with aws_ebs_volume.xfusion-volume,
│   on main.tf line 1, in resource "aws_ebs_volume" "xfusion-volume":
│    1: resource "aws_ebs_volume" "xfusion-volume" {
│ 
│ "snapshot_id": one of `size,snapshot_id` must be specified
╵

At least one of size or snapshot_id is required.







bob@iac-server ~/terraform via 💠 default ➜  vi main.tf

bob@iac-server ~/terraform via 💠 default ➜  terraform init
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "5.91.0"...
- Installing hashicorp/aws v5.91.0...
cat main


terrafo- Installed hashicorp/aws v5.91.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.

bob@iac-server ~/terraform via 💠 default ➜  cat main.tf 
resource "aws_ebs_volume" "xfusion-volume" {
 availability_zone = "us-east-1"
 size              = 2
 type              = "gp3"
}

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  terraform plan

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ebs_volume.xfusion-volume will be created
  + resource "aws_ebs_volume" "xfusion-volume" {
      + arn               = (known after apply)
      + availability_zone = "us-east-1"
      + encrypted         = (known after apply)
      + final_snapshot    = false
      + id                = (known after apply)
      + iops              = (known after apply)
      + kms_key_id        = (known after apply)
      + size              = 2
      + snapshot_id       = (known after apply)
      + tags_all          = (known after apply)
      + throughput        = (known after apply)
      + type              = "gp3"
    }

Plan: 1 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take
exactly these actions if you run "terraform apply" now.

bob@iac-server ~/terraform via 💠 default ➜  terraform apply -auto-approve

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ebs_volume.xfusion-volume will be created
  + resource "aws_ebs_volume" "xfusion-volume" {
      + arn               = (known after apply)
      + availability_zone = "us-east-1"
      + encrypted         = (known after apply)
      + final_snapshot    = false
      + id                = (known after apply)
      + iops              = (known after apply)
      + kms_key_id        = (known after apply)
      + size              = 2
      + snapshot_id       = (known after apply)
      + tags_all          = (known after apply)
      + throughput        = (known after apply)
      + type              = "gp3"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_ebs_volume.xfusion-volume: Creating...
aws_ebs_volume.xfusion-volume: Still creating... [10s elapsed]
aws_ebs_volume.xfusion-volume: Creation complete after 13s [id=vol-dfac18c826bfc2c4d]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  terraform plan
aws_ebs_volume.xfusion-volume: Refreshing state... [id=vol-dfac18c826bfc2c4d]

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_ebs_volume.xfusion-volume must be replaced
-/+ resource "aws_ebs_volume" "xfusion-volume" {
      ~ arn                  = "arn:aws:ec2:us-east-1::volume/vol-dfac18c826bfc2c4d" -> (known after apply)
      + availability_zone    = "us-east-1" # forces replacement
      ~ encrypted            = false -> (known after apply)
      ~ id                   = "vol-dfac18c826bfc2c4d" -> (known after apply)
      ~ iops                 = 3000 -> (known after apply)
      + kms_key_id           = (known after apply)
      - multi_attach_enabled = false -> null
      + snapshot_id          = (known after apply)
      - tags                 = {} -> null
      ~ tags_all             = {} -> (known after apply)
      ~ throughput           = 0 -> (known after apply)
        # (4 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take
exactly these actions if you run "terraform apply" now.

bob@iac-server ~/terraform via 💠 default ➜  terraform apply -auto-approve
aws_ebs_volume.xfusion-volume: Refreshing state... [id=vol-dfac18c826bfc2c4d]

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_ebs_volume.xfusion-volume must be replaced
-/+ resource "aws_ebs_volume" "xfusion-volume" {
      ~ arn                  = "arn:aws:ec2:us-east-1::volume/vol-dfac18c826bfc2c4d" -> (known after apply)
      + availability_zone    = "us-east-1" # forces replacement
      ~ encrypted            = false -> (known after apply)
      ~ id                   = "vol-dfac18c826bfc2c4d" -> (known after apply)
      ~ iops                 = 3000 -> (known after apply)
      + kms_key_id           = (known after apply)
      - multi_attach_enabled = false -> null
      + snapshot_id          = (known after apply)
      - tags                 = {} -> null
      ~ tags_all             = {} -> (known after apply)
      ~ throughput           = 0 -> (known after apply)
        # (4 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
aws_ebs_volume.xfusion-volume: Destroying... [id=vol-dfac18c826bfc2c4d]
aws_ebs_volume.xfusion-volume: Still destroying... [id=vol-dfac18c826bfc2c4d, 10s elapsed]
aws_ebs_volume.xfusion-volume: Destruction complete after 10s
aws_ebs_volume.xfusion-volume: Creating...
aws_ebs_volume.xfusion-volume: Still creating... [10s elapsed]
aws_ebs_volume.xfusion-volume: Creation complete after 10s [id=vol-7ae883830b929dff3]

Apply complete! Resources: 1 added, 0 changed, 1 destroyed.

bob@iac-server ~/terraform via 💠 default ➜  terraform apply -auto-approve
aws_ebs_volume.xfusion-volume: Refreshing state... [id=vol-7ae883830b929dff3]

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_ebs_volume.xfusion-volume must be replaced
-/+ resource "aws_ebs_volume" "xfusion-volume" {
      ~ arn                  = "arn:aws:ec2:us-east-1::volume/vol-7ae883830b929dff3" -> (known after apply)
      + availability_zone    = "us-east-1" # forces replacement
      ~ encrypted            = false -> (known after apply)
      ~ id                   = "vol-7ae883830b929dff3" -> (known after apply)
      ~ iops                 = 3000 -> (known after apply)
      + kms_key_id           = (known after apply)
      - multi_attach_enabled = false -> null
      + snapshot_id          = (known after apply)
      - tags                 = {} -> null
      ~ tags_all             = {} -> (known after apply)
      ~ throughput           = 0 -> (known after apply)
        # (4 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.
aws_ebs_volume.xfusion-volume: Destroying... [id=vol-7ae883830b929dff3]
aws_ebs_volume.xfusion-volume: Still destroying... [id=vol-7ae883830b929dff3, 10s elapsed]
aws_ebs_volume.xfusion-volume: Destruction complete after 10s
aws_ebs_volume.xfusion-volume: Creating...
aws_ebs_volume.xfusion-volume: Still creating... [10s elapsed]
aws_ebs_volume.xfusion-volume: Creation complete after 10s [id=vol-75c62f3ae669a4296]

Apply complete! Resources: 1 added, 0 changed, 1 destroyed.

bob@iac-server ~/terraform via 💠 default ➜  terraform list 
Terraform has no command named "list". Did you mean "test"?

To see all of Terraform's top-level commands, run:
  terraform -help


bob@iac-server ~/terraform via 💠 default ✖ terraform state list 
aws_ebs_volume.xfusion-volume

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  terrform destroy
bash: terrform: command not found

bob@iac-server ~/terraform via 💠 default ✖ terraform destroy
aws_ebs_volume.xfusion-volume: Refreshing state... [id=vol-75c62f3ae669a4296]

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_ebs_volume.xfusion-volume will be destroyed
  - resource "aws_ebs_volume" "xfusion-volume" {
      - arn                  = "arn:aws:ec2:us-east-1::volume/vol-75c62f3ae669a4296" -> null
      - encrypted            = false -> null
      - final_snapshot       = false -> null
      - id                   = "vol-75c62f3ae669a4296" -> null
      - iops                 = 3000 -> null
      - multi_attach_enabled = false -> null
      - size                 = 2 -> null
      - tags                 = {} -> null
      - tags_all             = {} -> null
      - throughput           = 0 -> null
      - type                 = "gp3" -> null
        # (4 unchanged attributes hidden)
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_ebs_volume.xfusion-volume: Destroying... [id=vol-75c62f3ae669a4296]







aws_ebs_volume.xfusion-volume: Still destroying... [id=vol-75c62f3ae669a4296, 10s elapsed]
aws_ebs_volume.xfusion-volume: Destruction complete after 10s

Destroy complete! Resources: 1 destroyed.

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  

bob@iac-server ~/terraform via 💠 default ➜  vi main.tf 

bob@iac-server ~/terraform via 💠 default ➜  terraform plan

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ebs_volume.xfusion-volume will be created
  + resource "aws_ebs_volume" "xfusion-volume" {
      + arn               = (known after apply)
      + availability_zone = "us-east-1"
      + encrypted         = (known after apply)
      + final_snapshot    = false
      + id                = (known after apply)
      + iops              = (known after apply)
      + kms_key_id        = (known after apply)
      + size              = 2
      + snapshot_id       = (known after apply)
      + tags              = {
          + "Name" = "xfusion-volume"
        }
      + tags_all          = {
          + "Name" = "xfusion-volume"
        }
      + throughput        = (known after apply)
      + type              = "gp3"
    }

Plan: 1 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take
exactly these actions if you run "terraform apply" now.

bob@iac-server ~/terraform via 💠 default ➜  terraform apply -auto-approve

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ebs_volume.xfusion-volume will be created
  + resource "aws_ebs_volume" "xfusion-volume" {
      + arn               = (known after apply)
      + availability_zone = "us-east-1"
      + encrypted         = (known after apply)
      + final_snapshot    = false
      + id                = (known after apply)
      + iops              = (known after apply)
      + kms_key_id        = (known after apply)
      + size              = 2
      + snapshot_id       = (known after apply)
      + tags              = {
          + "Name" = "xfusion-volume"
        }
      + tags_all          = {
          + "Name" = "xfusion-volume"
        }
      + throughput        = (known after apply)
      + type              = "gp3"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_ebs_volume.xfusion-volume: Creating...
aws_ebs_volume.xfusion-volume: Still creating... [10s elapsed]
aws_ebs_volume.xfusion-volume: Creation complete after 10s [id=vol-bb4e61e3c5886d5bd]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

bob@iac-server ~/terraform via 💠 default ➜  terraform plan
aws_ebs_volume.xfusion-volume: Refreshing state... [id=vol-bb4e61e3c5886d5bd]

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_ebs_volume.xfusion-volume must be replaced
-/+ resource "aws_ebs_volume" "xfusion-volume" {
      ~ arn                  = "arn:aws:ec2:us-east-1::volume/vol-bb4e61e3c5886d5bd" -> (known after apply)
      + availability_zone    = "us-east-1" # forces replacement
      ~ encrypted            = false -> (known after apply)
      ~ id                   = "vol-bb4e61e3c5886d5bd" -> (known after apply)
      ~ iops                 = 3000 -> (known after apply)
      + kms_key_id           = (known after apply)
      - multi_attach_enabled = false -> null
      + snapshot_id          = (known after apply)
        tags                 = {
            "Name" = "xfusion-volume"
        }
      ~ throughput           = 0 -> (known after apply)
        # (5 unchanged attributes hidden)
    }

Plan: 1 to add, 0 to change, 1 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take
exactly these actions if you run "terraform apply" now.

bob@iac-server ~/terraform via 💠 default ➜  











ssdsdsdsd


resource "aws_ebs_volume" "devops-volume" { 
  availability_zone = "us-east-1" 
  size = 2 
  type = "gp3"
  tags = {
    Name = "devops-volume"
 }
}












bob@iac-server ~/terraform via 💠 default ➜  vi main.tf

bob@iac-server ~/terraform via 💠 default ➜  vi main.tf^C

bob@iac-server ~/terraform via 💠 default ✖ vi main.tf^C

bob@iac-server ~/terraform via 💠 default ✖ terraform init
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "5.91.0"...
- Installing hashicorp/aws v5.91.0...
- Installed hashicorp/aws v5.91.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.

bob@iac-server ~/terraform via 💠 default ➜  terraform plan

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ebs_volume.datacenter-volume will be created
  + resource "aws_ebs_volume" "datacenter-volume" {
      + arn               = (known after apply)
      + availability_zone = "us-east-1a"
      + encrypted         = (known after apply)
      + final_snapshot    = false
      + id                = (known after apply)
      + iops              = (known after apply)
      + kms_key_id        = (known after apply)
      + size              = 2
      + snapshot_id       = (known after apply)
      + tags              = {
          + "Name" = "datacenter-volume"
        }
      + tags_all          = {
          + "Name" = "datacenter-volume"
        }
      + throughput        = (known after apply)
      + type              = "gp3"
    }

Plan: 1 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take
exactly these actions if you run "terraform apply" now.

bob@iac-server ~/terraform via 💠 default ➜  terraform apply -auto-approve

Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ebs_volume.datacenter-volume will be created
  + resource "aws_ebs_volume" "datacenter-volume" {
      + arn               = (known after apply)
      + availability_zone = "us-east-1a"
      + encrypted         = (known after apply)
      + final_snapshot    = false
      + id                = (known after apply)
      + iops              = (known after apply)
      + kms_key_id        = (known after apply)
      + size              = 2
      + snapshot_id       = (known after apply)
      + tags              = {
          + "Name" = "datacenter-volume"
        }
      + tags_all          = {
          + "Name" = "datacenter-volume"
        }
      + throughput        = (known after apply)
      + type              = "gp3"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_ebs_volume.datacenter-volume: Creating...
aws_ebs_volume.datacenter-volume: Still creating... [10s elapsed]
aws_ebs_volume.datacenter-volume: Creation complete after 12s [id=vol-e327e4793bcfeedce]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

bob@iac-server ~/terraform via 💠 default ➜  terraform plan
aws_ebs_volume.datacenter-volume: Refreshing state... [id=vol-e327e4793bcfeedce]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no
differences, so no changes are needed.

bob@iac-server ~/terraform via 💠 default ➜  

























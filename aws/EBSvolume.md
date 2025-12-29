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

























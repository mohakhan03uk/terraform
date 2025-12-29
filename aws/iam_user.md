# IAM USER
```
resource "aws_iam_user" "iamuser_john" {
  name = "iamuser_john"
}
```

```
C:\Users\mohakhan\Downloads\terraform_1.11.4_windows_386\iam_user>..\terraform.exe apply -auto-approve

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_user.iamuser_john will be created
  + resource "aws_iam_user" "iamuser_john" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "iamuser_john"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_iam_user.iamuser_john: Creating...
aws_iam_user.iamuser_john: Creation complete after 1s [id=iamuser_john]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

```


# IAM GROUP 
```
resource "aws_iam_group" "iamgroup_siva" {
  name = "iamgroup_siva"
}
```

```
C:\Users\mohakhan\Downloads\terraform_1.11.4_windows_386\iam_groups>..\terraform.exe apply -auto-approve

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_group.iamgroup_siva will be created
  + resource "aws_iam_group" "iamgroup_siva" {
      + arn       = (known after apply)
      + id        = (known after apply)
      + name      = "iamgroup_siva"
      + path      = "/"
      + unique_id = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_iam_group.iamgroup_siva: Creating...
aws_iam_group.iamgroup_siva: Creation complete after 1s [id=iamgroup_siva]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

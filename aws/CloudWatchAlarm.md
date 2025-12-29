Using Terraform, perform the following:

Task Details:
Create a CloudWatch alarm named devops-alarm.
The alarm should monitor CPU utilization of an EC2 instance.
Trigger the alarm when CPU utilization exceeds 80%.
Set the evaluation period to 5 minutes.
Use a single evaluation period.



resource "aws_cloudwatch_metric_alarm" "devops_alarm" {
  alarm_name          = "devops-alarm"
  alarm_description   = "Alarm when EC2 CPU exceeds 80%"

  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  statistic           = "Average"

  period              = 300
  evaluation_periods  = 1
  threshold           = 80
  comparison_operator = "GreaterThanThreshold"

  dimensions = {
    InstanceId = var.instance_id
  }
  treat_missing_data = "missing"
}

terraform init
terraform plan -var="instance_id=i-0abc123456789def0"
terraform apply -var="instance_id=i-0abc123456789def0"


bob@iac-server ~/terraform via 💠 default ➜  ll
total 24
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 ./
drwxr-x--- 1 bob bob 4096 Dec 29 15:07 ../
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf

bob@iac-server ~/terraform via 💠 default ➜  vi main.tf

bob@iac-server ~/terraform via 💠 default ➜  ll^C

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
╷
│ Error: Reference to undeclared input variable
│ 
│   on main.tf line 15, in resource "aws_cloudwatch_metric_alarm" "devops_alarm":
│   15:     InstanceId = var.instance_id
│ 
│ An input variable with the name "instance_id" has not been
│ declared. This variable can be declared with a variable
│ "instance_id" {} block.
╵

bob@iac-server ~/terraform via 💠 default ✖ vi main.tf

bob@iac-server ~/terraform via 💠 default ➜  terraform plan

Terraform used the selected providers to generate the following
execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # aws_cloudwatch_metric_alarm.devops_alarm will be created
  + resource "aws_cloudwatch_metric_alarm" "devops_alarm" {
      + actions_enabled                       = true
      + alarm_description                     = "Alarm when EC2 CPU exceeds 80%"
      + alarm_name                            = "devops-alarm"
      + arn                                   = (known after apply)
      + comparison_operator                   = "GreaterThanThreshold"
      + evaluate_low_sample_count_percentiles = (known after apply)
      + evaluation_periods                    = 1
      + id                                    = (known after apply)
      + metric_name                           = "CPUUtilization"
      + namespace                             = "AWS/EC2"
      + period                                = 300
      + statistic                             = "Average"
      + tags_all                              = (known after apply)
      + threshold                             = 80
      + treat_missing_data                    = "missing"
    }

Plan: 1 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so
Terraform can't guarantee to take exactly these actions if you
run "terraform apply" now.
bob@iac-server ~/terraform via 💠 d
bob@iac-server ~/terraform via 💠 d
bob@iac-server ~/terraform via 💠 d
bob@iac-server ~/terraform via 💠 d
bob@iac-server ~/terraform via 💠 d
bob@iac-server ~/terraform via 💠 d
bob@iac-server ~/terraform via 💠 d
bob@iac-server ~/terraform via 💠 defaul
bob@iac-server ~/terraform via 💠 defaul
bob@iac-server ~/terraform via 💠 default ➜  terraform apply -auto-approve 

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_cloudwatch_metric_alarm.devops_alarm will be created
  + resource "aws_cloudwatch_metric_alarm" "devops_alarm" {
      + actions_enabled                       = true
      + alarm_description                     = "Alarm when EC2 CPU exceeds 80%"
      + alarm_name                            = "devops-alarm"
      + arn                                   = (known after apply)
      + comparison_operator                   = "GreaterThanThreshold"
      + evaluate_low_sample_count_percentiles = (known after apply)
      + evaluation_periods                    = 1
      + id                                    = (known after apply)
      + metric_name                           = "CPUUtilization"
      + namespace                             = "AWS/EC2"
      + period                                = 300
      + statistic                             = "Average"
      + tags_all                              = (known after apply)
      + threshold                             = 80
      + treat_missing_data                    = "missing"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_cloudwatch_metric_alarm.devops_alarm: Creating...
aws_cloudwatch_metric_alarm.devops_alarm: Creation complete after 1s [id=devops-alarm]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

bob@iac-server ~/terraform via 💠 default ➜  cat main.tf 
resource "aws_cloudwatch_metric_alarm" "devops_alarm" {
  alarm_name          = "devops-alarm"
  alarm_description   = "Alarm when EC2 CPU exceeds 80%"

  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  statistic           = "Average"

  period              = 300
  evaluation_periods  = 1
  threshold           = 80
  comparison_operator = "GreaterThanThreshold"

  #dimensions = {
  #  InstanceId = var.instance_id
  #}

  treat_missing_data = "missing"
}


bob@iac-server ~/terraform via 💠 default ➜  

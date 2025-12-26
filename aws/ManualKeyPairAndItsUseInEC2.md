
# Understanding SSH Key Pairs for EC2 and Terraform

A clear, simple, complete guide explaining **what key pairs are**, **why they matter**, **how they are created**, and **how Terraform uses them**.

---

# 🔐 What Is an SSH Key Pair?
An **SSH key pair** is a **cryptographic authentication method** used to securely connect to Linux-based EC2 instances.

A key pair consists of:

| Key File | Where It Lives | Purpose |
|---------|----------------|----------|
| **Private Key (`mykey`)** | On *your* laptop only | Used to authenticate (prove your identity) |
| **Public Key (`mykey.pub`)** | Uploaded to AWS | Stored on the EC2 instance to allow your private key to log in |

You **never** upload the private key to AWS.

---

# 🧠 Why Do We Need Key Pairs for EC2?
When you launch a Linux EC2 instance, AWS does **not** set a password.

Instead, EC2 uses:
- your **public key** to generate an `authorized_keys` entry inside the instance
- your **private key** to authenticate SSH access

This makes remote access:
- passwordless
- encrypted
- secure
- resistant to brute-force attacks

If using **AWS SSM Session Manager**, key pairs are optional (explained later).

---

# 📁 Where Are Key Pair Files Stored?
When you run `ssh-keygen`, two files are created:

```
~/.ssh/mykey        ← PRIVATE key (keep safe)
~/.ssh/mykey.pub    ← PUBLIC key (upload to AWS)
```

Your `.pub` file is **safe to share** — it cannot be used to SSH without the matching private key.

---

# 🛠️ How To Generate Your Own Key Pair (Linux/macOS/WSL)
Run this command:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/mykey
```

This will:
- create `~/.ssh/mykey` (private key)
- create `~/.ssh/mykey.pub` (public key)

Set secure permissions:
```bash
chmod 600 ~/.ssh/mykey
```

---

# 🧩 How Terraform Uses SSH Keys
Terraform **never** creates private keys.
Terraform only uploads your **public key** (`.pub`) to AWS.

Terraform creates an AWS EC2 key pair using:

```hcl
resource "aws_key_pair" "this" {
  key_name   = "mykey"
  public_key = file("~/.ssh/mykey.pub")
}
```

AWS stores the public key and injects it into the EC2 instance.

Then Terraform launches the EC2 instance and attaches the key pair:

```hcl
resource "aws_instance" "ec2" {
  ami                         = "ami-0c02fb55956c7d316"
  instance_type               = "t2.micro"
  subnet_id                   = data.aws_subnets.default.ids[0]
  vpc_security_group_ids      = [aws_security_group.allow_ssh.id]
  key_name                    = aws_key_pair.this.key_name

  tags = { Name = "Terraform-EC2" }
}
```

This ensures your EC2 instance knows your public key.

---

# 🔌 How SSH Authentication Works
When you run:

```bash
ssh -i ~/.ssh/mykey ec2-user@<public-ip>
```

SSH performs this:
1. Your private key proves your identity
2. EC2 checks its authorized_keys list for your public key
3. Keys match → access granted

AWS **never sees your private key**.

---

# 🚫 What NOT To Do
### ❌ Do *NOT* upload your private key to AWS
### ❌ Do *NOT* commit keys to GitHub
### ❌ Do *NOT* share your private key with anyone
### ❌ Do *NOT* chmod 777 your key

Good key hygiene saves you from massive security risks.

---

# 🟩 Full Terraform Example (Key Pair + EC2)

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "default-for-az"
    values = ["true"]
  }
}

resource "aws_security_group" "allow_ssh" {
  name   = "allow_ssh"
  vpc_id = data.aws_vpc.default.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_key_pair" "this" {
  key_name   = "mykey"
  public_key = file("~/.ssh/mykey.pub")
}

resource "aws_instance" "ec2" {
  ami                         = "ami-0c02fb55956c7d316"
  instance_type               = "t2.micro"
  subnet_id                   = data.aws_subnets.default.ids[0]
  vpc_security_group_ids      = [aws_security_group.allow_ssh.id]
  key_name                    = aws_key_pair.this.key_name

  tags = {
    Name = "Terraform-EC2"
  }
}

output "public_ip" {
  value = aws_instance.ec2.public_ip
}
```

---

# 🟦 What If You Don’t Want SSH Keys? (SSM Session Manager)
You can skip key pairs completely if you use **AWS Systems Manager (SSM)**.

Benefits:
- No SSH port exposed
- No private key needed
- More secure

Requirements:
- IAM role with `AmazonSSMManagedInstanceCore`
- SSM agent installed (Amazon Linux & Ubuntu already include it)

Access using:
```
AWS Console → Systems Manager → Session Manager → Start Session
```

---

# 🎯 Summary
- You generate SSH keys **locally**, not AWS.
- Terraform uploads **only the public key** to AWS.
- EC2 stores that public key internally.
- You SSH with your **private** key.
- SSM can replace key pairs entirely.

---

# ✅ End of File

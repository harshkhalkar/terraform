<h1 align="center">
  <img src="https://www.vectorlogo.zone/logos/terraformio/terraformio-icon.svg" alt="Terraform Logo" width="50" />
  Terraform
</h1>

Terraform is an open-source **Infrastructure as Code (IaC)** tool developed by **HashiCorp**. It allows you to define, provision, and manage infrastructure across a wide range of cloud providers and services using a declarative configuration language called **HCL (HashiCorp Configuration Language)**.

**Other IaC Tools:**  
- CloudFormation (AWS)  
- Azure Bicep / ARM Template (Microsoft Azure)  
- Deployment Manager (GCP)

---

<h2 align="center">Benefits of Infrastructure as Code (IaC)</h2>

- **Automation** of infrastructure provisioning  
- **Version control** of infrastructure (since configs are files)  
- **Multi-cloud support** from a single tool (Terraform)  
- **Safe and predictable changes** with planning  
- **Reusability** via modules

---

<h2 align="center">Installation</h2>

## Linux (Debian/Ubuntu based)

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

gpg --no-default-keyring \
  --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
  --fingerprint

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com \
  $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt-get install terraform

# Verify installation
terraform --version
# Usage: terraform [global options] <subcommand> [args]

# Set up autocompletion
touch ~/.bashrc
terraform -install-autocomplete

# Open a new shell to activate autocompletion
```
Refer to [Terraform installation (Linux)](https://developer.hashicorp.com/terraform/install#linux) for guidance.

## Windows

1. Visit the official [Terraform Downloads Page](https://developer.hashicorp.com/terraform/downloads).
2. Select **Windows** → **AMD64** architecture.
3. Click to download the `.zip` file.

**Extract the ZIP File:**

1. Right-click the downloaded `.zip` file and choose **Extract All...**
2. Navigate to `C:\Program Files` & create a folder `Terraform`.
3. Extract it to a folder, e.g., `C:\Terraform`.

**Add Terraform to System PATH:**

1. Open the **Start Menu**, search for `Environment Variables`, and click **Edit the system environment variables**.
2. In the **System Properties** window, click **Environment Variables...**
3. Under **System variables**, find and select the `Path` variable, then click **Edit/Add**.
4. Click **New**, then enter: `C:\Program Files\Terraform`
5. 5. Click **OK** on all dialogs to save.

**Verify Installation:**

Open **Command Prompt** or **PowerShell**, then run:

```poweshell
terraform --version
```

### SetUp for VScode
1. Install Terraform extension in VS Code from the Extensions Marketplace.
2. Install AWS Toolkit as we're working with AWS.
3. Make sure shell/terminal recognizes the terraform command.

<h2 align="center">Configuring AWS Credentials in Terraform</h2>

## Hardcoding Credentials (Not Recommended) 
```bash
provider "aws" {
  region     = "us-east-1"
  access_key = "your-access-key"
  secret_key = "your-secret-key"
}
```

## Using AWS Profile from the AWS CLI/AWS Toolkit (Extension in VScode)
```bash
aws configure --profile my-profile   # default if not specified --profile
```
```bash
provider "aws" {
  region  = "us-east-1"
  profile = "my-profile"
}
```

## Environment Variables (Linux)
```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_REGION="us-east-1"
```

## IAM Role Attached to an EC2 Instance (EC2 Instance Profile)
```bash
provider "aws" {
  region = "us-east-1"
}
```
Terraform will automatically use the EC2 metadata service to get credentials.

More, MayBe Using a Secrets Manager (Terraform Vault) or Terraform Cloud.

<h2 align="center">main.tf (format)</h2>

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~< 6.0"
    }
  }
}

# Configure AWS Provider
provider "aws" {
  region = "us-east-1"
  # profile    = "default"       # Uncomment if multiple AWS accounts configured locally
  # access_key = ""
  # secret_key = ""
}

# Launch EC2 Instance
resource "aws_instance" "web" {
  ami           = "ami-020cba7c55df1f615"
  instance_type = "t2.micro"
  key_name      = "my-terraform-key"             # Attach instance to EC2 key pair
  vpc_security_group_ids = [aws_security_group.my-security-group-1.id]
  
  # count = 5                                   # Uncomment to launch multiple instances

  tags = {
    Name = "Instance-Name"
    # Name = "Instance-Name ${count.index}"      # Use when count specified (Instance-Name 0, 1, 2, ...)
    # Name = "Instance-Name ${count.index + 1}"  # Starts indexing from 1
  }
}

# Create Security Group
resource "aws_security_group" "my-security-group-1" {
  name        = "terraform-sg"
  description = "Allow SSH & HTTP"

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"             # -1 means all protocols
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Generate TLS Private Key
resource "tls_private_key" "example_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

# Create AWS EC2 Key Pair using generated public key
resource "aws_key_pair" "example_key_pair" {
  key_name   = "my-terraform-key"
  public_key = tls_private_key.example_key.public_key_openssh
}

# Save private key to local file
resource "local_file" "example_private_key" {
  content         = tls_private_key.example_key.private_key_pem
  filename        = "my-terraform-key.pem"
  file_permission = "0400"
}
```
---

## `main.tf` Overview

This file defines the infrastructure to be deployed using Terraform. Below is a breakdown of each major section:

---

### `terraform` Block

Specifies the required provider:

- **AWS** from `hashicorp/aws`
- **Version constraint:** `~< 6.0`  
  (any version less than 6.0, but compatible with previous minor versions)

---

### AWS Provider Configuration

- **Region:** `us-east-1`
- **Supports different authentication methods:**
  - Default profile (via AWS CLI)
  - Access & secret keys *(not recommended for production)*
  - Environment variables
  - IAM role if running on EC2

---

### EC2 Instance: `aws_instance.web`

Launches an EC2 instance with:

- **AMI ID:** `ami-020cba7c55df1f615`
- **Instance type:** `t2.micro`
- **Key pair:** `my-terraform-key`
- **Security Group:** Attached via `vpc_security_group_ids`

**Tags:**

- Used to name the instance.
- Optional logic for multiple instances using `count` and `${count.index}`

---

### Security Group: `aws_security_group.my-security-group-1`

- **Name:** `terraform-sg`

**Ingress (Inbound Rules):**

- Port **22** (SSH) — open to all
- Port **80** (HTTP) — open to all

**Egress (Outbound Rules):**

- All traffic allowed (`0.0.0.0/0`, all protocols)

---

### Key Pair Generation

1. **`tls_private_key.example_key`**  
   - Generates a 2048-bit RSA private key locally.

2. **`aws_key_pair.example_key_pair`**  
   - Uploads the **public key** to AWS and creates a Key Pair named `my-terraform-key`.

3. **`local_file.example_private_key`**  
   - Saves the **private key** to a `.pem` file with secure permissions:
     - **Filename:** `my-terraform-key.pem`
     - **Permissions:** `0400` (read-only for owner)

---

<h2 align="center">Common Commands</h2>

```bash
terraform --version    # to check version of terraform

terraform init         # sets up the working directory and downloads necessary plugins

terraform plan         # shows what actions Terraform will take to match the desired state. OR Dry Run

terraform apply        # creates or updates the infrastructure.
terraform apply --auto-approve   # will not prompt for verification / it will approve automatically

terraform destroy      # cleanly tear down infrastructure / delete resources that were created
terraform destroy --auto-approve

terraform fmt          # Format Configuration Files Aligns indentation (Organizes spacing, Improves readability, Does not change the functionality)

terraform validat      # Checks whether your Terraform files are syntactically valid and internally consistent
```

<h2 align="center">Terraform Files</h2>

## State Files

Terraform uses these to keep track of the infrastructure it manages.

- `terraform.tfstate` - Records the current state of your infrastructure
- `terraform.tfstate.backup` - Auto-generated backup of the last known good state

## Configuration Files

These are your primary infrastructure definition files written in HCL (HashiCorp Configuration Language).

- `main.tf` - Contains core resource definitions (e.g., EC2, VPC, etc).
- `variables.tf` - Declares input variables used across modules or in `main.tf`.
- `outputs.tf` - Defines outputs you want Terraform to display or export.
- `terraform.tfvars` - Supplies values to variables declared in `variables.tf` / `main.tf`.

## Hidden Files and Directories

- `.terraform/` - Stores downloaded providers and modules.
- `.terraform.lock.hcl` - Dependency lock file to ensure consistent provider versions.

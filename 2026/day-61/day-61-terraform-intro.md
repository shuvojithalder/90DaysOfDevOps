## Day 61 – Terraform on AWS EC2 (Intro Walkthrough)

This document explains everything done in this mini‑project: from installing Terraform on an EC2 instance to creating and destroying AWS resources (S3 bucket and EC2 instance) with Terraform.

---

## 1. Environment Setup on EC2

### 1.1 Launch EC2 in `us-east-1`

- Launched an EC2 instance in `**us-east-1**`.
- Attached an **IAM role** (instance profile) with at least:
  - `AmazonEC2FullAccess`
  - `AmazonS3FullAccess`

This lets Terraform running inside the instance call AWS APIs without access keys.

### 1.2 Install Terraform (and basic tools) on Ubuntu

On the Ubuntu EC2 instance:

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install unzip, curl, etc.
sudo apt install -y unzip curl

# Download Terraform (example version)
TERRAFORM_VERSION="1.7.0"
curl -LO "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip"

unzip "terraform_${TERRAFORM_VERSION}_linux_amd64.zip"
sudo mv terraform /usr/local/bin/

# Verify
terraform version
```

Optional: install AWS CLI (for debugging and verification):

```bash
sudo apt install -y awscli
aws --version
```

### 1.3 AWS CLI / Region Configuration

Because the instance uses an IAM role, you don’t need to configure access keys, but you **do** need the correct region:

`~/.aws/config`:

```ini
[default]
region = us-east-1
```

Additionally, you can force the region in the shell:

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
```

This ensures the AWS SDK and Terraform agree on the region.

---

## 2. Terraform Project Structure

Project directory: this repo (`day1`).

- `terraform.tf` – required providers and Terraform version
- `provider.tf` – AWS provider configuration (region)
- `main.tf` – actual AWS resources (S3 bucket + EC2 instance)

### 2.1 `terraform.tf`

Defines Terraform core version and AWS provider:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

### 2.2 `provider.tf`

Configures Terraform to use AWS in `**us-east-1**`:

```hcl
provider "aws" {
  region = "us-east-1"
}
```

> Important: Region mismatches between the provider, AWS config, and actual endpoints cause errors like  
> `AuthorizationHeaderMalformed: The authorization header is malformed; the region 'us-east-1' is wrong; expecting 'eu-west-1'`.  
> That error was fixed by ensuring:
>
> - EC2 is in `us-east-1`
> - `~/.aws/config` has `region = us-east-1`
> - `provider "aws" { region = "us-east-1" }`

### 2.3 `main.tf`

Defines:

- An S3 bucket (for learning, not yet used as a backend here)
- An EC2 instance

```hcl
resource "aws_s3_bucket" "my-bucket" {
  bucket = "dev-state-bucket-1234567890"
  tags = {
    Name = "dev-state-bucket"
  }
}

resource "aws_instance" "my-instance" {
  ami           = "ami-0c3389a4fa5bddaad"
  instance_type = "t2.micro"
  tags = {
    Name = "TerraWeek-Day1"
  }
}
```

Notes:

- S3 bucket names are **global**, so the bucket name must be unique.
- There was a typo earlier: `aaws_instance` instead of `aws_instance`. That caused Terraform to look for a non‑existent provider `hashicorp/aaws`. Fixing the resource type to `aws_instance` resolved the `hashicorp/aaws` provider error.

---

## 3. Terraform Workflow on EC2

### 3.1 Initialize the Project

From the project directory:

```bash
terraform init
```

What this does:

- Downloads the required provider (`hashicorp/aws`).
- Prepares the `.terraform` directory.

If there are typos in resource types (like `aaws_instance`), Terraform will try to resolve a fake provider (`hashicorp/aaws`) and fail with:

> `provider registry registry.terraform.io does not have a provider named registry.terraform.io/hashicorp/aaws`

That was fixed by correcting the resource type to `aws_instance`.

### 3.2 Validate Configuration (Optional)

```bash
terraform validate
```

This checks HCL syntax and basic configuration.

### 3.3 See the Execution Plan

```bash
terraform plan
```

You should see:

- `+` create S3 bucket `dev-state-bucket-1234567890`
- `+` create EC2 instance `aws_instance.my-instance`

If there are region errors (e.g., “expecting `eu-west-1`”), double‑check:

- `provider.tf` region
- `~/.aws/config`
- `AWS_REGION` / `AWS_DEFAULT_REGION`

All must match `us-east-1` for this lab.

### 3.4 Apply (Create Resources)

```bash
terraform apply
```

Terraform will:

- Show the plan again.
- Ask for confirmation:
  ```text
  Do you want to perform these actions?
    Terraform will perform the actions described above.
    Only 'yes' will be accepted to approve.

  Enter a value:
  ```

Type `yes` and press Enter.

After a successful apply:

- S3 bucket `dev-state-bucket-1234567890` exists in `us-east-1`.
- EC2 instance `aws_instance.my-instance` is created in `us-east-1`.

You can verify in the AWS console:

- **S3 → Buckets**
- **EC2 → Instances**

---

## 4. Troubleshooting Highlights

### 4.1 Region Mismatch (`AuthorizationHeaderMalformed`)

Error seen:

> `AuthorizationHeaderMalformed: The authorization header is malformed; the region 'us-east-1' is wrong; expecting 'eu-west-1'`

Root cause:

- AWS request was signed for `us-east-1`, but some configuration/endpoint expected `eu-west-1`.

Fix:

- Confirm EC2 region (should be `us-east-1` for this lab).
- Align:
  - `provider "aws" { region = "us-east-1" }`
  - `~/.aws/config` → `[default] region = us-east-1`
  - `AWS_REGION` and `AWS_DEFAULT_REGION` (if set) → `us-east-1`

### 4.2 Wrong Provider Name (`hashicorp/aaws`)

Error seen:

> `provider registry.registry.terraform.io does not have a provider named registry.terraform.io/hashicorp/aaws`

Root cause:

- Typo in resource type: `aaws_instance` instead of `aws_instance`.
- Terraform inferred a provider named `aaws`.

Fix:

- Rename resource type to `aws_instance`.
- Keep `required_providers` pointing to `hashicorp/aws`.

---

## 5. Destroying the Infrastructure (`terraform destroy`)

To tear down everything Terraform created in this project:

```bash
terraform destroy
```

Terraform will:

- Show which resources will be deleted:
  - `aws_s3_bucket.my-bucket`
  - `aws_instance.my-instance`
- Ask for confirmation:
  ```text
  Do you really want to destroy all resources?
  Enter a value:
  ```

Type `yes` to proceed.

After a successful destroy:

- The S3 bucket and EC2 instance defined in `main.tf` are gone.
- The local state file (`terraform.tfstate`) still exists locally unless manually removed.

---

## 6. Summary of the Day 61 Lab

- **Environment**: Terraform running directly on an **EC2 instance** using an **IAM role** (no access keys).
- **Region**: Target region is `**us-east-1`**, with provider, AWS config, and environment all aligned.
- **Resources created**:
  - One **S3 bucket** for learning (and potentially a backend later).
  - One **EC2 instance** (`t2.micro`) with a specific AMI and tags.
- **Key lessons**:
  - Always align AWS region across **Terraform**, **AWS CLI config**, and **env vars**.
  - Resource type names control which provider Terraform expects (`aws_…` → `hashicorp/aws`).
  - Use `terraform init → plan → apply → destroy` as the standard workflow.

This README captures the full flow from **installation** on EC2 to **creating and destroying** AWS infrastructure with Terraform for this introductory lab.

---

## 7. Inspecting Terraform State (`terraform.tfstate`)

Terraform tracks everything it creates in a **state file** named `terraform.tfstate` in your project directory.

### 7.1 Open and Read `terraform.tfstate`

- After `terraform apply`, open `terraform.tfstate` in your editor.
- It is a **JSON** document that includes:
  - The Terraform version and serial number
  - Information about each resource (`aws_s3_bucket.my-bucket`, `aws_instance.my-instance`)
  - Attributes for each resource (IDs, ARNs, tags, etc.)

You normally don’t edit this file by hand; it is for Terraform’s internal tracking, but reading it helps you understand how Terraform maps configuration to real AWS objects.

### 7.2 Useful State Commands

Run these in the project directory (on the EC2 instance):

```bash
terraform show
```

- **What it does**: Prints a **human‑readable view of the current state**.
- You will see all managed resources and their attributes (similar to the JSON in `terraform.tfstate`, but formatted nicely).

```bash
terraform state list
```

- **What it does**: Lists **all resources** Terraform currently manages in this state.
- Example output:
  - `aws_s3_bucket.my-bucket`
  - `aws_instance.my-instance`

```bash
terraform state show aws_s3_bucket.my-bucket
```

- **What it does**: Shows a **detailed view of a specific resource** in state.
- You will see bucket name, ARN, region, tags, and other attributes Terraform knows about `aws_s3_bucket.my-bucket`.

```bash
terraform state show aws_instance.my-instance
```

- **What it does**: Shows **all stored attributes** for the EC2 instance resource.
- You will see instance ID, AMI, instance type, subnet, security groups, tags, etc. as Terraform understands them.

These commands help you inspect and debug the current Terraform state without manually editing `terraform.tfstate`.

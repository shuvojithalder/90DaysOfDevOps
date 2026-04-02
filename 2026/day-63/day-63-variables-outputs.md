## Day 63 – Variables, tfvars, outputs, and functions (this project)

This README documents `**variables.tf**`, **dev and prod `.tfvars`**, `**terraform output**`, **variable precedence**, **built-in functions**, and how `**variable`**, `**locals**`, `**output**`, and `**data**` differ.

### Project files


| File               | Role                                                                       |
| ------------------ | -------------------------------------------------------------------------- |
| `terraform.tf`     | Terraform and AWS provider version constraints                             |
| `provider.tf`      | AWS provider `region` from `var.region`                                    |
| `variables.tf`     | All `variable` blocks                                                      |
| `main.tf`          | VPC, subnet, IGW, routes, security group, EC2, S3 — uses `var.*`, `locals` |
| `output.tf`        | `output` blocks — values after apply                                       |
| `terraform.tfvars` | **Dev / default** — auto-loaded                                            |
| `prod.tfvars`      | **Prod** — use with `-var-file`                                            |


---

## `variables.tf` (all variables and types)

Full file:

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "project_name" {
  type = string
}

variable "environment" {
  type    = string
  default = "dev"
}

variable "allowd_ports" {
  type    = list(string)
  default = ["22", "80", "443"]
}

variable "extra_tags" {
  type    = map(string)
  default = {}
}
```


| Variable        | Type in this project | Role                                    |
| --------------- | -------------------- | --------------------------------------- |
| `region`        | `string`             | AWS region for the provider             |
| `vpc_cidr`      | `string`             | VPC CIDR block                          |
| `instance_type` | `string`             | EC2 instance size                       |
| `project_name`  | `string`             | Required name prefix (no default)       |
| `environment`   | `string`             | e.g. `dev`, `prod`                      |
| `allowd_ports`  | `list(string)`       | TCP ports for the security group        |
| `extra_tags`    | `map(string)`        | Extra resource tags merged in `main.tf` |


**Other types you can use elsewhere:** `number`, `bool`, `set(...)`, `object({ ... })`, `tuple([...])`. This lab uses **string**, **list**, and **map**.

---

## Both `.tfvars` files: dev and prod

Terraform does **not** merge dev + prod at once: you either rely on auto-loaded `**terraform.tfvars`** (dev) or pass `**prod.tfvars**` on the CLI for prod.

### Dev — `terraform.tfvars` (auto-loaded)

Used for normal runs (`terraform plan` / `apply` with no `-var-file`). No `vpc_cidr` override here, so `**vpc_cidr**` stays the default from `variables.tf` (`10.0.0.0/16`).

```hcl
project_name   = "terraweek"
environment    = "dev"
instance_type  = "t2.micro"
```

### Prod — `prod.tfvars` (explicit)

Pass explicitly: `terraform plan -var-file="prod.tfvars"`. Overrides `**vpc_cidr**` for a different network; subnet CIDR in `**main.tf**` is derived with `**cidrsubnet(var.vpc_cidr, 8, 1)**`.

```hcl
project_name   = "terraweek"
environment    = "prod"
instance_type  = "t3.small"
vpc_cidr       = "10.1.0.0/16"
# Subnet: cidrsubnet("10.1.0.0/16", 8, 1) → 10.1.1.0/24
```

---

## Screenshot: outputs after `terraform apply`

After a successful apply, capture the printed **Outputs:** section at the end of the log, or run `terraform output` / `terraform output -json` and screenshot that.

**Suggested image for this lab** (save your own PNG next to this README and fix the path):

```markdown
![Terraform outputs after apply](terraform-outputs-after-apply.png)
```

Example of what the CLI shows (values are examples only):

```text
Outputs:

instance_id = "i-0abc123def456789"
instance_public_dns = "ec2-1-2-3-4.compute-1.amazonaws.com"
instance_public_ip = "1.2.3.4"
security_group_id = "sg-0123456789abcdef"
subnet_id = "subnet-0abc123def456789"
vpc_id = "vpc-0abc123def456789"
```

Replace with your real IDs from `terraform output`.

---

## Variable precedence (lowest → highest) with examples

When the **same** variable is set in multiple places, **higher rows win**.


| Priority    | Source                      | Example                                                                                                   |
| ----------- | --------------------------- | --------------------------------------------------------------------------------------------------------- |
| 1 (lowest)  | `default` in `variable { }` | `environment` defaults to `"dev"` if nothing else sets it.                                                |
| 2           | `TF_VAR_<name>`             | `export TF_VAR_environment=staging` → `staging`, unless overridden below.                                 |
| 3           | `terraform.tfvars`          | `environment = "dev"` overrides `TF_VAR_environment` for that variable.                                   |
| 4           | `terraform.tfvars.json`     | Same idea as row 3, JSON file.                                                                            |
| 5           | `*.auto.tfvars`             | e.g. `staging.auto.tfvars` loaded in filename order.                                                      |
| 6 (highest) | `-var` / `-var-file`        | `terraform apply -var-file="prod.tfvars"` → prod values override rows 1–5 for variables set in that file. |


**Worked examples (this repo):**

1. **Only defaults** — No `terraform.tfvars`, no env, no CLI: `region` = `"us-east-1"` from `variables.tf`.
2. **Dev tfvars** — `terraform.tfvars` has `environment = "dev"`: that beats `default` and beats `TF_VAR_environment` if both were set.
3. **Prod file** — `terraform apply -var-file="prod.tfvars"` sets `environment = "prod"` and `vpc_cidr = "10.1.0.0/16"`; those beat auto-loaded `terraform.tfvars` for the same keys.
4. **One-off override** — `terraform plan -var="instance_type=t2.nano"` wins over `terraform.tfvars` and `prod.tfvars` for `instance_type` (CLI tier).

Official reference: [Variable definition precedence](https://developer.hashicorp.com/terraform/language/values/variables#variable-definition-precedence).

---

## Five built-in functions (most useful in this project)


| #   | Function         | Why it’s useful here                                                                               |
| --- | ---------------- | -------------------------------------------------------------------------------------------------- |
| 1   | `**merge`**      | Combine `common_tags` with per-resource `Name` tags: `merge(local.common_tags, { Name = "..." })`. |
| 2   | `**join**`       | Build `name_prefix` without long interpolation: `join("-", [var.project_name, var.environment])`.  |
| 3   | `**cidrsubnet**` | Derive the public subnet `/24` from `var.vpc_cidr` (`/16`) with fixed sizing.                      |
| 4   | `**lookup**`     | Map `environment` to compliance tier or prod `instance_type` without many `if` chains.             |
| 5   | `**format**`     | Build ARNs or other strings with `%s` placeholders (e.g. S3 ARNs in outputs if you add them).      |


More examples: see **Terraform built-in functions (quick reference)** at the end of this file.

---

## `variable` vs `locals` vs `output` vs `data`


|                        | **Variable (`variable`)**                                  | **Local (`locals`)**                                   | **Output (`output`)**                                              | **Data (`data`)**                                                                         |
| ---------------------- | ---------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| **Purpose**            | Inputs **into** the module                                 | Named **intermediate** values inside the module        | Values **out** of the module (for humans, scripts, parent modules) | **Read** existing cloud/API objects without **creating** them                             |
| **Who sets the value** | Operator: tfvars, `-var`, `TF_VAR_*`, or `default` in code | You: expressions in `locals { }`                       | Terraform: `value =` from resources, vars, locals                  | Terraform: provider query (e.g. latest AMI)                                               |
| **Referenced as**      | `var.name`                                                 | `local.name`                                           | `terraform output` / `module.child.output_name`                    | `data.type.name.attribute`                                                                |
| **In this repo**       | `variables.tf`                                             | `main.tf` `locals` (e.g. `name_prefix`, `common_tags`) | `output.tf`                                                        | *Not used* — you could add e.g. `data "aws_ami" "ubuntu" { }` to resolve an AMI by filter |


**Mental model:** **Variables** = knobs from outside. **Locals** = shorthand and logic inside one module. **Outputs** = published results. **Data** = look up something that already exists (read-only).

---

## Terraform output

**Outputs** are declared with `output` blocks. They expose values after `**terraform apply`** for the CLI, scripts, CI, or parent modules.

### Syntax (typical)

```hcl
output "instance_public_ip" {
  description = "Public IPv4 of the EC2 instance"
  value       = aws_instance.my_instance.public_ip
}
```

Official reference: [Output values](https://developer.hashicorp.com/terraform/language/values/outputs).

### Outputs in this project (`output.tf`)


| Output name           | Value                                     |
| --------------------- | ----------------------------------------- |
| `vpc_id`              | `aws_vpc.my_vpc.id`                       |
| `subnet_id`           | `aws_subnet.my_subnet.id`                 |
| `instance_id`         | `aws_instance.my_instance.id`             |
| `instance_public_ip`  | `aws_instance.my_instance.public_ip`      |
| `instance_public_dns` | `aws_instance.my_instance.public_dns`     |
| `security_group_id`   | `aws_security_group.my_security_group.id` |


`instance_public_dns` is only set when the VPC has `**enable_dns_hostnames = true**` and the instance has a public hostname.

### CLI commands

```bash
terraform output
terraform output instance_public_ip
terraform output -json
```

---

## Commands you used (recap)

1. `**terraform plan**` — Loads `terraform.tfvars` automatically.
2. `**terraform plan -var-file="prod.tfvars"**` — Prod values from CLI tier.
3. `**terraform plan -var="instance_type=t2.nano"**` — One-off override.
4. `**export TF_VAR_environment="staging"**` then `**terraform plan**` — Note: `terraform.tfvars` still wins over `TF_VAR_*` if it sets `environment`.

---

## Where variables are used

- `**provider.tf**` — `region = var.region`
- `**main.tf**` — `var.vpc_cidr`, `var.allowd_ports`, `var.extra_tags`, `locals` (`merge`, `join`, `cidrsubnet`, `lookup`, etc.)

---

## Terraform built-in functions (quick reference)

Full list: [Functions overview](https://developer.hashicorp.com/terraform/language/functions).

#### String


| Function | Example                                  | Result                     |
| -------- | ---------------------------------------- | -------------------------- |
| `upper`  | `upper("terraweek")`                     | `"TERRAWEEK"`              |
| `lower`  | `lower("TERRAWEEK")`                     | `"terraweek"`              |
| `join`   | `join("-", ["terra", "week", "2026"])`   | `"terra-week-2026"`        |
| `format` | `format("arn:aws:s3:::%s", "my-bucket")` | `"arn:aws:s3:::my-bucket"` |


#### Collection


| Function | Example                                                  | Result       |
| -------- | -------------------------------------------------------- | ------------ |
| `length` | `length(["a", "b", "c"])`                                | `3`          |
| `lookup` | `lookup({ dev = "t2.micro", prod = "t3.small" }, "dev")` | `"t2.micro"` |
| `merge`  | `merge({ a = "1" }, { b = "2" })`                        | combined map |
| `toset`  | `toset(["a", "b", "a"])`                                 | unique set   |


#### Networking


| Function     | Example                           | Result          |
| ------------ | --------------------------------- | --------------- |
| `cidrsubnet` | `cidrsubnet("10.0.0.0/16", 8, 1)` | `"10.0.1.0/24"` |


#### Example `locals` block

```hcl
locals {
  example_join   = join("-", [var.project_name, var.environment])
  example_lookup = lookup({ dev = "t2.micro", prod = "t3.small" }, var.environment, "t2.micro")
  example_cidr   = cidrsubnet(var.vpc_cidr, 8, 1)
}
```


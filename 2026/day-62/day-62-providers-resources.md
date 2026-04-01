## Full `main.tf` with comments

```hcl
# Create a custom VPC for all our resources
resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"

  # Tag helps identify the VPC in AWS console
  tags = {
    Name = "TerraWeek-VPC"
  }
}

# Public subnet inside the custom VPC
resource "aws_subnet" "my_subnet" {
  # Implicit dependency: subnet waits for aws_vpc.my_vpc via this reference
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "TerraWeek-Public-Subnet"
  }
}

# Internet Gateway attached to the custom VPC
resource "aws_internet_gateway" "my_internet_gateway" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "TerraWeek-Internet-Gateway"
  }
}

# Route table to send all outbound traffic to the Internet Gateway
resource "aws_route_table" "my_route_table" {
  vpc_id = aws_vpc.my_vpc.id

  # Default route to the internet for 0.0.0.0/0
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my_internet_gateway.id
  }

  tags = {
    Name = "TerraWeek-Route-Table"
  }
}

# Associate the route table with the public subnet
resource "aws_route_table_association" "my_route_table_association" {
  # Implicit deps on both subnet and route table via references
  subnet_id      = aws_subnet.my_subnet.id
  route_table_id = aws_route_table.my_route_table.id
}

# Security group for the EC2 instance (SSH + HTTP open to the world)
resource "aws_security_group" "my_security_group" {
  name        = "TerraWeek-SG"
  description = "TerraWeek-Security-Group"
  vpc_id      = aws_vpc.my_vpc.id

  # Allow SSH from anywhere (lab/demo only!)
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow HTTP from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow all outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "TerraWeek-SG"
  }
}

# EC2 instance in the public subnet, using the security group above
resource "aws_instance" "my_instance" {
  ami           = "ami-0ec10929233384c7f"
  instance_type = "t2.micro"

  # Place instance in our public subnet
  subnet_id = aws_subnet.my_subnet.id

  # Ensure it gets a public IP for internet access/SSH
  associate_public_ip_address = true

  # Attach the VPC security group we created
  vpc_security_group_ids = [aws_security_group.my_security_group.id]

  # Ensure a new instance is created before the old one is destroyed
  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "TerraWeek-Server"
  }
}

# S3 bucket for application logs
resource "aws_s3_bucket" "app_logs" {
  bucket = "terraweek-app-logs-bucket"

  # Explicit dependency: wait for EC2 instance even though not referenced
  depends_on = [aws_instance.my_instance]

  tags = {
    Name = "TerraWeek-App-Logs"
  }
}
```

## Screenshots

- **Terraform apply output**  
 <img width="1252" height="590" alt="Task5 2" src="https://github.com/user-attachments/assets/d565ddc9-01d6-4adb-a594-f2238094a5fe" />

- **VPC and related resources in AWS console**  
VPC and resources
  <img width="1360" height="668" alt="Task4" src="https://github.com/user-attachments/assets/7a3080d8-48a7-4d90-bf8b-7e81813e2ca6" />

  
## Dependency graph (text)

```text
aws_vpc.my_vpc
  ├─ aws_subnet.my_subnet
  │    └─ aws_instance.my_instance
  │          └─ aws_s3_bucket.app_logs (explicit depends_on)
  ├─ aws_internet_gateway.my_internet_gateway
  │    └─ aws_route_table.my_route_table
  │          └─ aws_route_table_association.my_route_table_association
  └─ aws_security_group.my_security_group
        └─ aws_instance.my_instance
```

## Implicit vs explicit dependencies (in my own words)

- **Implicit dependencies**: Terraform automatically works out the order of resources when one resource references another in its arguments. For example, `aws_subnet.my_subnet` refers to `aws_vpc.my_vpc.id`, so Terraform knows it must create the VPC first and then the subnet, without me writing any `depends_on`. Any reference like `resource_a.foo.id` inside `resource_b` creates this kind of implicit dependency.
- **Explicit dependencies**: Sometimes there is a required order that is *not* visible from the arguments. In those cases I can use `depends_on` to tell Terraform about the relationship. In this configuration, `aws_s3_bucket.app_logs` does not need anything from the EC2 instance, but I still want it created only after `aws_instance.my_instance`, so I add `depends_on = [aws_instance.my_instance]` to force that ordering.


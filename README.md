## Additional Terraform Configuration: Remote State, VPC Networking and EC2 Variables

### 1. S3 Remote Backend with State Locking

The project uses an Amazon S3 bucket to store Terraform state remotely.

```hcl
terraform {
  backend "s3" {
    bucket       = "ram-terraform-state-2026-001"
    key          = "terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true
  }
}
```

**Configuration explanation**

| Argument | Description |
|---|---|
| `backend "s3"` | Configures Amazon S3 as the Terraform backend |
| `bucket` | S3 bucket that stores the state file |
| `key` | Location of the state object inside the bucket |
| `region` | AWS region containing the bucket |
| `use_lockfile = true` | Enables native S3 state locking using a `.tflock` object |

**How it works**

1. Terraform initializes the S3 backend.
2. The state file is stored remotely in the configured S3 bucket.
3. Terraform acquires a lock during supported state operations to help prevent concurrent modifications.
4. After the operation completes, Terraform releases the lock.

**Important:** Create the S3 bucket before running `terraform init`. The bucket must be in `us-east-1`, and the AWS identity Terraform uses must have the required S3 permissions for state and lock objects.

Recommended protections include S3 versioning, encryption, restricted bucket access, and a recovery procedure for state files.

Initialize the backend:

```bash
terraform init
```

If you are migrating existing local state to this backend, Terraform may ask to copy the state to S3. Review the prompt carefully before proceeding.

### 2. AWS VPC Configuration

The project creates a VPC with CIDR block `10.0.0.0/16`.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "MY-VPC"
  }
}
```

The VPC provides an isolated network for AWS resources.

### 3. Public and Private Subnets

The configuration creates two subnets in different Availability Zones.

| Subnet | CIDR block | Availability Zone | Purpose |
|---|---|---|---|
| Public subnet | `10.0.1.0/24` | `us-east-1a` | Resources requiring public network connectivity |
| Private subnet | `10.0.2.0/24` | `us-east-1b` | Resources that should not have direct public internet routing |

The public subnet has:

```hcl
map_public_ip_on_launch = true
```

This allows AWS to assign public IPv4 addresses automatically to supported instances launched in that subnet, subject to the instance configuration.

**Important:** A subnet is not public merely because of its name. Its route table must provide a route to an Internet Gateway, and the instance must have suitable public addressing and security rules.

### 4. Internet Gateway

```hcl
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "MY-IGW"
  }
}
```

The Internet Gateway attaches to the VPC and allows resources with appropriate routes and public addressing to communicate with the internet.

### 5. Public Route Table and Internet Route

The public route table contains a default route to the Internet Gateway.

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "Public-Route-Table"
  }
}

resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}
```

The route `0.0.0.0/0` matches IPv4 destinations not covered by a more specific route. Traffic matching this default route is directed through the Internet Gateway.

The public subnet is associated with the route table:

```hcl
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

### 6. Private Route Table

```hcl
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "Private-Route-Table"
  }
}

resource "aws_route_table_association" "private" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_route_table.private.id
}
```

The private subnet uses a separate route table without a default internet route in this configuration.

A NAT Gateway and a corresponding route would be needed if private instances require outbound IPv4 internet access. NAT Gateways can incur additional charges.

### 7. EC2 Input Variables

The configuration declares input variables for EC2:

```hcl
variable "ami_id" {
  description = "AMI ID for EC2"
  type        = string
  default     = "ami-0b6d9d3d33ba97d99"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}
```

| Variable | Purpose |
|---|---|
| `ami_id` | Specifies the Amazon Machine Image used to launch the instance |
| `instance_type` | Specifies the instance's compute and memory configuration |

The AMI ID must exist in the selected region and match the intended CPU architecture. Confirm that `ami-0b6d9d3d33ba97d99` is valid for `us-east-1` before deploying.

**Note:** Declaring these variables does not create an EC2 instance. An `aws_instance` resource is also required.

### 8. Terraform Deployment Workflow

Run these commands from the Terraform root directory:

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

Inspect the plan before confirming deployment.

To inspect the resources managed by the current state:

```bash
terraform state list
terraform output
```

To remove resources managed by the current state after testing:

```bash
terraform plan -destroy
terraform destroy
```

### 9. Key Learning Outcomes

- Configure an S3 remote backend for Terraform state.
- Understand native S3 state locking.
- Create AWS VPC networking with public and private subnets.
- Configure an Internet Gateway and route tables.
- Use multiple Availability Zones for subnet placement.
- Declare reusable EC2 input variables.
- Understand the difference between defining variables and creating resources.
- Apply secure state-management and infrastructure-cleanup practices.

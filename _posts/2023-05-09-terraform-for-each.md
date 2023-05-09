---
layout: single
title: How to use Terraform for_each correctly
toc: true
tags: terraform
---

Terraform language provides the **for_each** meta-argument to create many resources from a key-value data structure. However, if not used properly, the for_each meta-argument can produce unexpected results.

In this post, we'll see a Terraform module using for_each incorrectly and eventually we'll see how to fix it.

# Terraform module using for_each

Assume you are writing a Terraform module to provision AWS EC2 instances on different availability zones. The modules reads the availability zone list from the user and use it to create the EC2 instances using a `for_each` meta-argument.

The `variables.tf` file looks like:
```hcl
variable "instance_type" {
  type        = string
  description = "Type of instance"
}

variable "regions" {
  type        = list(string)
  description = "List of regions where to deploy instances"
}
```

The main.tf file looks like:
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}

resource "aws_instance" "ubuntu" {
  for_each = { for k, region in var.regions : k => region }

  ami                         = data.aws_ami.ubuntu.id
  associate_public_ip_address = true
  availability_zone           = each.value
  instance_type               = var.instance_type

  tags = {
    Name = "instance-${each.key}"
  }
}
```

Use this module in an example configuration file, such as `examples/simple/main.tf`:
```hcl
terraform {
  required_version = ">= 1.0"
}

module "test" {
  source        = "../.."
  instance_type = "t2.micro"
  regions       = ["us-east-1a", "us-east-1b"]
}
```

If you run `terraform init` and `terraform apply` for the first time, of course Terraform will create 2 instances.
Below the output of terraform apply:
```text
...
module.test.aws_instance.ubuntu["0"]: Creation complete after 35s [id=i-0451049885ba98fc8]
module.test.aws_instance.ubuntu["1"]: Creation complete after 35s [id=i-00f4a6556c7084f3f]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

# A subtle trap using Terraform for_each 
What happens if you remove `"us-east-1a"` from the `regions` list and run `terraform apply` again?
The example configuration using the module becomes:
```hcl
terraform {
  required_version = ">= 1.0"
}

module "test" {
  source        = "../.."
  instance_type = "t2.micro"
  regions       = ["us-east-1b"]
}
```

The `terraform plan` output:
```text
...

Plan: 1 to add, 0 to change, 2 to destroy.
```

We want to destroy just the first instance. Why does Terraform want to destroy both resources instead?

# Understanding for_each key index

Our Terraform module gives the following map to the for_each meta-argument:
```hcl
for_each = { for k, region in var.regions : k => region }
```
The indexes of the list `var.regions` are the keys of the map, that Terraform converts to strings: "0", "1", etc.

The elements of the list `var.regions` are the values of the map: "us-east-1a", "us-east-1b".

When we remove the first element from the list `var.regions`, Terraform wants to destroy the resource with key "0". Since the list now has one elements, Terraform plans to recreate the resource with key "1" so that the new resource has the new key "0".

We would like Terraform to not change the second instance. To do so, we have to give a unique name to the map key instead of an incremental number.

Run `terraform destroy` to clean the environment and revert the regions list in `examples/simple/main.tf`.

# Fix the module

To fix the module, change the for_each line in main.tf file with the following:
```hcl
...
for_each = { for k, region in var.regions : region => region }
...
```

Run `terraform apply`:
```text
...
module.test.aws_instance.ubuntu["us-east-1a"]: Creation complete after 35s [id=i-081bc2e3c99087238]
module.test.aws_instance.ubuntu["us-east-1b"]: Creation complete after 45s [id=i-07ce30d140b4bf7ce]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```
Compared to before, the resource keys are "us-east-1a" and "us-east-1b" instead of "0" and "1".

Now remove `"us-east-1a"` from the `regions` list in `examples/simple/main.tf` file and run `terraform plan` again:
```text
...
Terraform will perform the following actions:

  # module.test.aws_instance.ubuntu["us-east-1a"] will be destroyed
  # (because key ["us-east-1a"] is not in for_each map)
...
Plan: 0 to add, 0 to change, 1 to destroy.
```

Hurrah! Terraform destroy the resource related to the region we removed while keeping the other resource.

# Conclusions

Terraform for_each meta-argument is a powerful tool to create many resources of the same type but remember to use a unique index key.

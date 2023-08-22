---
layout: single
title: Avoid name conflicts in Terraform workspaces
toc: true
tags: terraform
---

Terraform workspaces are a feature of some backends to associate many terraform states to the same configuration. It is convenient to have a workspace where you can try changes to infrastructure. When the changes have been tested, you can switch to the production workspace and apply the changes.

Since the configuration files are the same for all workspaces, there may be conflicts among the names of the cloud resources.

For instance, you want to create a AWS security group named `allow_http`. You switch to the **lab workspace** and apply the configuration. Now you have a security group named “allow_http” in your AWS account.

You switch to the **prod** **workspace** and apply the terraform configuration again. However, Terraform complains that the security group `allow_http` already exists!

When you use terraform workspaces, you should pay attention to clashes among the names of cloud resources.

At the time of writing this post, the latest version of Terraform is 1.5.5.

# Create a terraform resource with a fixed name

You have a Terraform configuration that creates a security group:

```yaml
resource "aws_security_group" "allow_http" {
  name        = "allow_http"
  description = "Allow HTTP inbound traffic"

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
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
```

## Create the resources in the workspace “dev”

First you create the terraform workspace `dev` and then you apply the configuration:

```console
$ terraform workspace new dev
$ terraform apply
```

Copy the id of the `security_group` somewhere because it will be needed later:

```console
$ terraform state show aws_security_group.allow_http
...
id = "sg-0a89544d88354a70b"
...
```

## Create the resources in the workspace “prod”

Then you want to create the resources in the new terraform workspace `prod`:

```console
$ terraform workspace new prod
```

If you make a `terraform plan`, terraform wants to create the security group again:

```console
$ terraform plan
...
Plan: 1 to add, 0 to change, 0 to destroy.
```

If you try to apply the changes, it fails because the resource already exists:

```console
$ terraform apply
...
Error: creating Security Group (allow_http): InvalidGroup.Duplicate: The security group 'allow_http' already exists for VPC 'vpc-0688682e4898523a4'
...
```

# Solution 1: Import resources already existing

First solution is to import the existing resource in this workspace as well:

```console
$ terraform import aws_security_group.allow_http sg-0a89544d88354a70b
```

However, in larger terraform configurations may be complex to import all resources that cannot be duplicated in other workspaces.

# Solution 2: Use terraform.workspace in resource names

The second approach is to use the `terraform.workspace` variable in resource names to avoid duplicate errors:

```yaml
resource "aws_security_group" "allow_http" {
  name        = "allow_http_${terraform.workspace}"
  description = "Allow HTTP inbound traffic"

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
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
```

# Conclusions

Terraform workspaces are a convenient feature to manage many terraform states related to the same configuration. However, you should use the `terraform.workspace` variable to avoid conflicts with the resources created by other workspaces.

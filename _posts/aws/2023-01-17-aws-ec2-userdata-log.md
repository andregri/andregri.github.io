---
layout: single
title: Debugging EC2 user_data script
tags: aws
---

AWS EC2 instances can be initialized easily at startup using a custom script in the `user_data` field.
Our instances will be ready to use and have all tools installed.
It is a great advantage when the initialization script is easy, because we don't need to use separate tool like Ansible.
However, `user_data` can be tricky because it is not so friendly to debug in case of failures.

For instance, if you are writing a `user_data` from a Terraform configuration, remember to remove any extra space.
Otherwise the script will fail quietly. In Terraform, you can write the commands inline using the **Heredoc string**:
```tf
resource "aws_instance" "node1" {
  ...

  user_data = <<EOF
#!/bin/bash -x
echo "hello world"
EOF
  
  ...
}
```

If you want to keep your Terraform file neat, you can use the **indented heredoc string**, that will remove leading spaces from each line:
```tf
resource "aws_instance" "node1" {
  ...

  user_data = <<-EOF
  #!/bin/bash -x
  echo "hello world"
  EOF
  
  ...
}
```

Remember to set the `-x` option because it will allow to see all steps and output in the EC2 instance logs!

Once the EC2 instance has been initialized (all status checks are passed), you can verify what the `user_data script` did in the log file located at `/var/log/cloud-init-output.log`.

---
layout: single
title: Kubernetes The Hard Way - Write Ansible inventory from template with Terraform
tags: kthw terraform
---

Once the instances are provisioned to AWS using a Terraform script, I need to configure them using Ansible. To tell Ansible the remote targets to configure, I wrote an **inventory** file that lists the IP addresses and the hostnames. Each time I created the AWS sandbox, the IP addresses changed and I needed to copy-paste them on the inventory file. Since this list was growing too fast, I opted for using a template and letting Terraform do this job.

Terraform provider **local** allows to write files locally provided the file path and the file content. The file path is `ansible/inventory`. The file content can be generated at run time using a template that contains some variables. So I created the file `iac/templates/inventory.tftpl`.

```ini
[controllers]
controller-0    ansible_host=${controller-0-public-ip}    private_ip=${controller-0-private-ip}     private_dns=${controller-0-private-dns}
controller-1    ansible_host=${controller-1-public-ip}    private_ip=${controller-1-private-ip}     private_dns=${controller-1-private-dns}

[controllers:vars]
ansible_connection=ssh
ansible_user=ubuntu
ansible_ssh_private_key_file=${ssh-private-key}

[workers]
worker-0    ansible_host=${worker-0-public-ip}      private_ip=${worker-0-private-ip}   private_dns=${worker-0-private-dns}
worker-1    ansible_host=${worker-1-public-ip}      private_ip=${worker-1-private-ip}   private_dns=${worker-1-private-dns}

[workers:vars]
ansible_connection=ssh
ansible_user=ubuntu
ansible_ssh_private_key_file=${ssh-private-key}
```

The variables are filled with the values of the resources in the Terraform configuration using the function `templatefile` that takes two arguments: the template file path and a dictionary with the values to be assigned to the variables.

```tf
resource "local_file" "inventory" {
  content  = templatefile(
    "${path.module}/templates/inventory.tftpl",
    { 
        controller-0-public-ip   = module.ec2_instance[0].public_ip,
        controller-0-private-ip  = module.ec2_instance[0].private_ip,
        controller-0-private-dns = module.ec2_instance[0].private_dns,
        controller-1-public-ip   = module.ec2_instance[1].public_ip,
        controller-1-private-ip  = module.ec2_instance[1].private_ip,
        controller-1-private-dns = module.ec2_instance[1].private_dns,
        worker-0-public-ip       = module.ec2_workers[0].public_ip,
        worker-0-private-ip      = module.ec2_workers[0].private_ip,
        worker-0-private-dns     = module.ec2_workers[0].private_dns,
        worker-1-public-ip       = module.ec2_workers[1].public_ip,
        worker-1-private-ip      = module.ec2_workers[1].private_ip,
        worker-1-private-dns     = module.ec2_workers[1].private_dns,
        ssh-private-key          = "kthw.pem"
    }
  )
  filename = "${path.module}/../ansible/inventory"
}
```

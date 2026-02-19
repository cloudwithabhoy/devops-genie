# IaC Cheatsheet

> Quick-reference commands and concepts for Terraform and Ansible.

## Terraform

### Workflow Commands
```bash
terraform init          # initialise working directory
terraform validate      # validate configuration
terraform plan          # preview changes
terraform apply         # apply changes
terraform apply -auto-approve
terraform destroy       # destroy resources
terraform output        # show outputs
terraform state list    # list resources in state
terraform state show <resource>
terraform import <resource> <id>   # import existing resource
terraform fmt           # format code
```

### HCL Basics
```hcl
# Provider
provider "aws" {
  region = "us-east-1"
}

# Resource
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags = { Name = "web-server" }
}

# Variable
variable "region" {
  default = "us-east-1"
}

# Output
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

## Ansible

### CLI Commands
```bash
ansible all -m ping -i inventory.ini
ansible-playbook playbook.yml -i inventory.ini
ansible-playbook playbook.yml --check   # dry run
ansible-playbook playbook.yml --tags "install"
ansible-vault encrypt secrets.yml
ansible-vault decrypt secrets.yml
ansible-galaxy role install <role>
```

### Playbook Structure
```yaml
---
- name: Configure web server
  hosts: webservers
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: true
```

## Common Patterns

<!-- Add Terraform module patterns, Ansible role patterns here -->

---
title: How to use Terraform and Ansible to raise a Jenkins Server on DigitalOcean
date: 2019-05-20
hero: "/img/black-textile-41949.jpg"
excerpt: Start by installing Ansible and Terraform. On MacOSX you can do that using Homebrew, or simply by downloading the binaries from the official websites.
timeToRead: 10
authors:
  - Adrian Gheorghe

---
[Published to Medium](https://medium.com/@adrian.gheorghe.dev/how-to-use-terraform-and-ansible-to-raise-a-jenkins-server-on-digitalocean-15246b687666)


> [HashiCorp Terraform](https://www.terraform.io/) enables you to safely and predictably create, change, and improve infrastructure. It is an open source tool that codifies APIs into declarative configuration files that can be shared amongst team members, treated as code, edited, reviewed, and versioned.

> In computing, [Ansible](https://www.ansible.com/) is an open-source software provisioning, configuration management, and application-deployment tool. It runs on many Unix-like systems, and can configure both Unix-like systems as well as Microsoft Windows. It includes its own declarative language to describe system configuration.

I’m not going to get into the advantages of having both your project infrastructure and configuration in code here, but [Terraform](https://www.terraform.io/) and [Ansible](https://www.ansible.com/) are great tools for doing both of these.

### Installing Ansible and Terraform

Start by installing Ansible and Terraform. On MacOSX you can do that using Homebrew, or simply by downloading the binaries from the official websites.

```bash
brew install terraform  
brew install ansible
```

After you have Terraform and Ansible accessible, install [terraform-inventory](https://github.com/adammck/terraform-inventory). This is a Go application that generates a dynamic inventory file from your Terraform state. This way, you will be able to use the Terraform newly created server information as your Ansible host.

You might want to install this manually because homebrew does not always have the latest version available.

```bash
brew install terraform-inventory 
```

### DigitalOcean Api keys and SSH key

The next step would be to create a DigitalOcean Read Write Token for Terraform to use when connecting to the DigitalOcean api. You can do this in your Account > Api > Personal access tokens. Keep the token safe and well away any versioned code.

In addition to this, in order to store the Terraform state in a secure manner we will be using DigitalOcean Spaces. You will need to create a Space and a Spaces Access Key that Terraform will be using to store its State.

> More information on this can be found in this excellent post by Joseph D. Marhee: [https://medium.com/@jmarhee/digitalocean-spaces-as-a-terraform-backend-b761ae426086](https://medium.com/@jmarhee/digitalocean-spaces-as-a-terraform-backend-b761ae426086)

In addition to the access key and token you will need a ssh key that will be used to connect to the droplets.

```bash
ssh-keygen -t rsa -b 4096
```

Upload the public key to DigitalOcean and retrieve the key fingerprint, it will be required when you run terraform plan or apply.

### Terraform Files

Create a provider.tf file containing the DigitalOcean backend configuration and set it up with the DataCenter your droplets and spaces will be in.
```hcl-terraform
variable "digitalocean_token" {}  
variable "digitalocean_ssh_fingerprint" {}  
  
provider "digitalocean" {  
    token = "${var.digitalocean_token}"  
    version = "~> 1.0"  
}  
  
terraform {  
  backend "s3" {  
    endpoint = "ams3.digitaloceanspaces.com"  
    region = "us-west-1"  
    key = "terraform.tfstate"  
    skip_requesting_account_id = true  
    skip_credentials_validation = true  
    skip_get_ec2_platforms = true  
    skip_metadata_api_check = true  
  }  
}
```
Now create a jenkins.tf file containing the following configuration.
```hcl-terraform
resource "digitalocean_tag" "jenkins" {  
    name = "example:jenkins"  
}  
  
# Create a new Droplet using the SSH key  
resource "digitalocean_droplet" "example_jenkins" {  
  name     = "example-jenkins"  
  image    = "ubuntu-18-04-x64"  
  region   = "ams3"  
  size     = "s-1vcpu-1gb"  
  ipv6 = true  
  private_networking = true  
  ssh_keys = ["${var.digitalocean_ssh_fingerprint}"]  
  tags = ["${digitalocean_tag.jenkins.name}"]  
}
```
This will create a 1CPU 1GB RAM Ubuntu 18.04 64b Droplet with the name example-jenkins. The Droplet will be created in the Amsterdam DigitalOcean DataCenter, but you can change this to whatever one you need. The ssh key attached to the droplet will be the one you pass in when asking Terraform to create the droplets.

Assuming you want to point a domain or subdomain to the new droplet, you can use Terraform for that as well. The following configuration will add example.org as a domain in your DigitalOcean DNS management and then add a jenkins.example.org A record pointing to that IP.

```hcl-terraform
resource "digitalocean_domain" "exampleorg" {  
    name = "example.org"  
}  
  
resource "digitalocean_record" "exampleorg" {  
    name = "jenkins"  
    type = "A"  
    domain = "${digitalocean_domain.exampleorg.name}"  
    value = "${digitalocean_droplet.example_jenkins.ipv4_address}"  
}
```
The last bit of infrastructure configuration we will want to add is to set the droplet firewall to only allow specific ports open. (22 for SSH, 80 and 443 for web access and 8080 which is the default port Jenkins runs on)

```hcl-terraform
resource "digitalocean_firewall" "jenkins" {  
    name = "firewall-jenkins"  
    droplet_ids = ["${digitalocean_droplet.example_jenkins.id}"]  
  
    inbound_rule = [  
        {  
            protocol = "tcp"  
            port_range = "22"  
        },  
        {  
            protocol = "tcp"  
            port_range = "80"  
        },  
        {  
            protocol = "tcp"  
            port_range = "443"  
        },  
        {  
            protocol = "tcp"  
            port_range = "8080"  
        }  
    ]  
}
```
### Init Terraform, Plan and Apply

Initialize terraform with the correct access key
```bash
terraform init \  
   -backend-config="access_key=AAABBBCCCDDDEEE" \  
   -backend-config="secret_key=AAABBBCCCDDDEEE" \  
   -backend-config="bucket=AAABBBCCCDDDEEE"

# Run terraform plan to check what the script will actually do.  
terraform plan

# Run terraform apply to create the droplet  
terraform apply
```
After this hopefully you should have an Ubuntu droplet set up to accept SSH connections and the domain jenkins.example.org pointing to it. Let’s move on to provisioning it.

### Provisioning the droplet with Ansible

In order to provision the droplet we will need to run an Ansible playbook.

Create a an Ansible requirements.yml file
```yaml
---  
- src: geerlingguy.git  
- src: geerlingguy.java  
- src: geerlingguy.jenkins  
- src: hispanico.nginx-revproxy  
- src: hispanico.letsencrypt-nginx-revproxy  
- src: geerlingguy.mysql  
- src: geerlingguy.docker  
- src: geerlingguy.ansible  
- src: geerlingguy.pip
```
Create a jenkins.yml playbook either in this directory.

```yaml
---  
- hosts: example_jenkins  
  become: true  
  any_errors_fatal: true  
  gather_facts: no  
  pre_tasks:  
    - name: 'install python2'  
      raw: sudo apt-get -y install python  
    - name: Set the java_packages variable (Ubuntu).  
      set_fact:  
        java_packages:  
          - openjdk-8-jdk  
    - name: update cache  
      apt:  
        update_cache: true  
    - name: Install apache2-utils  
      apt:  
        name: apache2-utils  
        state: present  
  
- hosts: example_jenkins  
  become: true  
  any_errors_fatal: true  
  gather_facts: yes  
  vars:  
    jenkins_hostname: jenkins.example.org  
  roles:  
    - role: geerlingguy.java  
    - role: geerlingguy.jenkins  
      become: yes  
      vars:  
        jenkins_admin_username: admin  
        jenkins_admin_password: admin  
        jenkins_package_state: present  
    - role: hispanico.nginx-revproxy  
    - role: hispanico.letsencrypt-nginx-revproxy  
      vars:  
        nginx_revproxy_sites:  
          jenkins.example.org:  
            domains:  
              - jenkins.example.org  
            upstreams:  
              - { backend_address: 127.0.0.1, backend_port: 8080 }  
            letsencrypt: true  
            letsencrypt_email: 'test@example.org'  
            ssl: true  
    - role: geerlingguy.pip  
      vars:  
        ansible_install_method: pip  
        ansible_install_version_pip: "2.7.0"  
    - role: geerlingguy.ansible  
    - role: geerlingguy.docker
```
### HTTPS

The Jenkins service exposes the UI on port 8080 by default. In order to access Jenkins via HTTPS the Ansible playbook contains an nginx proxy running on the same server. These excellent roles add an nginx proxy service on the same droplet and generate a SSL Certificate using Let’s Encrypt.

[**hispanico/ansible-nginx-revproxy**](https://github.com/hispanico/ansible-nginx-revproxy)

### Run the Ansible provisioning script:

After terraform has finished deploying the droplet, run the following commands.

```bash
ansible-galaxy install -r requirements.yml

ansible-playbook --inventory=`which terraform-inventory` --private-key=id_rsa --user=root jenkins.yml
```
In order to provision the droplet, you should be able to add a provisioner declaration in your jenkins.tf file like the one below. I personally have not been able to get this to work using terraform-inventory so I’ve kept the scripts completely separate.
```hcl-terraform

provisioner "local-exec" {  
    command = <<EOT  
      ls -al  
      ansible-galaxy install -r requirements.yml  
      ansible-playbook --inventory=`which terraform-inventory` --private-key=id_rsa --user=root jenkins.yml  
    EOT  
}
```
When the Ansible script finishes, you should have Jenkins installed in the new Droplet.

Accessing [http://jenkins.example.org](http://jenkins.example.org):8080 should give you access to the Jenkins UI.
Accessing [https://jenkins.example.org](http://jenkins.example.org) should give you the Jenkins UI via https.


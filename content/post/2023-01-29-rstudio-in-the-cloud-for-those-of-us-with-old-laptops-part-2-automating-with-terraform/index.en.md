---
title: 'Rstudio in the cloud for those of us with old laptops part 2: automating with
  terraform'
author: novica
date: '2023-01-29'
slug: rstudio-in-the-cloud-for-those-of-us-with-old-laptops-part-2-automating-with-terraform
categories:
  - R
tags:
  - Rstudio
  - aws
  - ec2
  - terraform
subtitle: ''
summary: ''
authors: [novica]
lastmod: '2023-01-29T14:44:30+01:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

## Setting the scene for automation

In the previous post I wrote about how to [spin up a EC2 instance with Rstudio server](/post/rstudio-in-the-cloud-for-those-of-us-with-old-laptops), so some of the more computational heavy `R` processing can be moved on AWS infrastructure. 

Let's say you've done this. Kept the EC2 for some time and then decided to terminate it, since after all, even stopped, it still incurs some costs. Then, some time after that, you need to spin up a new instance, and have to go through all of the manual clicking through the AWS console described before. 

Enter `terraform`. It is a tool to write infrastructure as code, or more descriptively, as human readable instructions to define resources that can be run on any cloud provider. And since these instructions live in text files, you can have them versioned with `git`, and keep track of any changes over time. I think this is awesome.

Additionally, `terraform` works with all major cloud providers, so if you prefer to use something else insted of `aws` you can adjust.

## Prerequisites for trying out terraform for configuring Rstudio server 

Two things need to be done before we can see `terraform` in action. 

1. [Install](https://aws.amazon.com/cli/) `aws cli`;
2. [Install](https://developer.hashicorp.com/terraform/downloads) `terraform`.

I am not going to go into details here because different operating systems might have different steps on how to do it, so I suggest you follow the official documentation for your system. 

That these have been successfully installed you can check with `aws --version` and `terraform --version` in your preferred shell.

Additionally, `aws` needs to be configured by typing: `aws configure`. Then we have to enter the AWS Access Key ID, AWS Secret Access Key, and Default Region Name for the IAM user. If you don't have IAM user set up, you really should. Here is a [guide from AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html)).

## Writing the first ever terraform configuration

The `Terraform` documentation is pretty good, so we are not really writing anything new, [but copying from there](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-build). In a 
folder called `rstudio-terraform` or something more appropriate, you can create a `main.tf` file and paste the following:

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
  }

  required_version = ">= 1.2.0"
}

provider "aws" {
  region  = "us-west-2"
}

resource "aws_instance" "app_server" {
  ami           = "ami-830c94e3"
  instance_type = "t2.micro"

  tags = {
    Name = "ExampleAppServerInstance"
  }
}
```

Its worth understanding the above specification and the best place to learn more about  the three blocks `terraform`, `provider`, and `resource` is in the docs. 

The changes that I made were:

1. to add a different tag, so I changed `ExampleAppServerInstance` to `RstudioTerraform`, and

2. to change the AMI. Since I was using Ubuntu before, I want to keep that, so I head out to [Ubuntu Cloud Image Finder](https://cloud-images.ubuntu.com/locator/) and find the AMI code for `22.04` which is `ami-03e08697c325f02ab`.

Additionally I added a security group and a key name. If you did the the previous  manual steps you should have these ready so just name them in the configuration.  Finally, to test that things work, I am requesting an output of the public IP of 
the resource that is going to be created. So my final `main.tf` file looks like this:

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
  }

  required_version = ">= 1.2.0"
}

provider "aws" {
  region = "eu-central-1"
}

resource "aws_instance" "app_server" {
  ami           = "ami-03e08697c325f02ab"
  instance_type = "t2.micro"
  security_groups = ["THE_NAME_OF_THE_SECURITY_GROUP"]
  key_name = "THE NAME OF THE KEY"
  tags = {
    Name = "RstudioTerraform"
  }
}

output "my-public-ip"{
       value= aws_instance.app_server.public_ip
}
```

Now, once this is saved, in the folder that holds the `main.tf` file, in the terminal we run `terraform init`, followed by `terraform plan`, and finally `terraform apply`. 

The first two commands should be instantaneous, and the last one should take maybe 20 seconds to complete. 

The EC2 instance should show up in the AWS console, and you can verify that the IP address that was printed on the terminal is the same one that the instance has in the Console.

This only gets us half way. We still need to do bunch of stuff before we have Rstudio  server running. But for now you can do `terraform destroy` and see how the EC2 instance is being terminated. Repeating `terraform apply` will create a new instance, and `terraform destroy` will destroy it again.

## Extending the terraform configuration to set up Rstudio server

Next, we need to run all those other commands in Ubuntu that update packages, install `R` and `Rstudio server`, create a user, and maybe something more. 

It is possible to keep the whole configuration in one file, so adding code to `main.tf` would not be a problem. However, it seems more convenient to have multiple files that hold logical parts together. In `R` terms think of it as package that has multiple functions in different `R` files.

So, create a new `tf` file, maybe called `remote.tf` since the code in there will do things on the remote EC2 instance.

The contents will be as follows:

```
resource "null_resource" "remote"{

connection {
       type = "ssh"
       user = "ubuntu"
       private_key = file("/full/path/to/the/key.pem")
       host  = aws_instance.app_server.public_ip
}

provisioner "remote-exec" {
         inline = [
                       
                    # update indices
                    "sudo apt update -qq",
                    # install two helper packages we need
                    "sudo apt install --no-install-recommends software-properties-common dirmngr",
                    # add the signing key (by Michael Rutter) for these repos
                    # To verify key, run gpg --show-keys /etc/apt/trusted.gpg.d/cran_ubuntu_key.asc 
                    # Fingerprint: E298A3A825C0D65DFD57CBB651716619E084DAB9
                    "wget -qO- https://cloud.r-project.org/bin/linux/ubuntu/marutter_pubkey.asc | sudo tee -a /etc/apt/trusted.gpg.d/cran_ubuntu_key.asc",
                    # add the R 4.0 repo from CRAN -- adjust 'focal' to 'groovy' or 'bionic' as needed
                    "sudo add-apt-repository --yes 'deb https://cloud.r-project.org/bin/linux/ubuntu $(lsb_release -cs)-cran40/'",
                    "sudo apt install --yes --no-install-recommends r-base",
                    "sudo add-apt-repository --yes ppa:c2d4u.team/c2d4u4.0+",
                    "sudo apt install --yes --no-install-recommends r-cran-tidyverse",
                    "sudo apt-get install --yes  gdebi-core",
                    "wget https://download2.rstudio.org/server/jammy/amd64/rstudio-server-2022.12.0-353-amd64.deb",
                    "sudo gdebi -n rstudio-server-2022.12.0-353-amd64.deb"

                  ]
  }

provisioner "file" {
    source      = "/local/path/rstudio-terraform/rserver.conf"
    destination = "/home/ubuntu/rserver.conf"
    }


provisioner "remote-exec" {
  inline = [
    "sudo mv /home/ubuntu/rserver.conf /etc/rstudio/rserver.conf",
    "sudo systemctl restart rstudio-server.service ",
    # setup the rstudio user
    "sudo groupadd rstudio-users",
    "sudo useradd -m -s /bin/bash -p $(perl -e 'print crypt($ARGV[0], 'password')' 'YOUR_PASSWORD') rstudio",
    "sudo usermod -a -G rstudio-users rstudio"
  ]
  }

}
```


That's quite a lot of code. Let's go through it step by step. 

## Understanding the sections in the additional .tf configuration file

The file begins with `resource "null_resource" "remote"{`. I don't know why exactly this is added. The [documentation](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource) 
is kind of unclear to me, but a lot of places mention this as the approach (and it turned out it works).

Next, it is the `connection` part which I think is straightforward. We are just telling `terraform` 
to connect to this newly created instance using `ssh` with the key we provide.

Next, the `remote-exec` section is telling `terraform` to execute bunch of commands 
on the `remote`. Clever :). The commands that we are executing are copied from 
[the official instructions for installing Ubuntu packages for R](`https://cran.rstudio.com/bin/linux/ubuntu/`). 
The only changes made is adding `--yes` to `apt install` and `-n` to `gdebi`,  because 
we want these to be executed without asking something like `are you sure you want to install...`, 
and because there is no way to answer this promnt (at least as far as I could see) once `terraform` is ran.

Next, the `file` section, uploads the `rserver.conf` uploads the configuration for `Rstudio server` 
on the EC2 instance. If you remember from the [previous post](/post/rstudio-in-the-cloud-for-those-of-us-with-old-laptops) 
we need to configure which users can be able to access `Rstudio server`. `Terraform` works in two steps when uploading files. First we upload the file to the home directory of the user that is logged in, then we move that from one to another place on the remote EC2.

The `rserver.conf` should have these two lines:

```
# users allowed to access rstudio
auth-required-user-group=rstudio-users
```

The next `remote-exec` section then completes the setup with coping the file locally 
on the EC2, restarting the service and creating the proper user and group, also 
following the documentation linked in the previous post.

Save the `remote.tf` and go through the `plan` and `apply` steps once more. You 
should see the new instance created in about two minutes (because installing some of 
the packages will take time). 

Then log in to the Rstudio to verify that everything works as expected. Nice!

## Final thoughts

There are some other stuff to discuss here. Running `git init` will convert the 
folder to a git repository (which can even be added to Github) -- just make sure 
to have any keys (if you have them in the same folder) listed in the `.gitignore` 
file. And the [management](https://spacelift.io/blog/terraform-state) of the `tfstate` 
file is a topic in it self, especially if you plan to share the resource within your organization.


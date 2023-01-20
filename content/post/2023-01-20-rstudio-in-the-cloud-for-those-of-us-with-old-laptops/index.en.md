---
title: 'Rstudio in the cloud for those of us with old laptops '
author: novica
date: '2023-01-20'
slug: rstudio-in-the-cloud-for-those-of-us-with-old-laptops
categories:
  - R
tags:
  - Rstudio
  - aws
  - ec2
subtitle: ''
summary: ''
authors: [novica]
lastmod: '2023-01-20T14:38:47+01:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---


I use a Thinkpad from 2015 for my day to day work, and you can imagine things are getting slower as time goes by. Recently I had to test some computation in an R package I am developing and my computer froze multiple times. I keep seeing the memory usage indicator in Rstudio going red as soon as something more demanding is ran. 

So I decided to use Rstudio serve on a EC2 instance for some of the more demanding tasks, and this post is mostly to keep track of the steps I did, and have a handy reference in the future. 

I will assume that there is no need to explain how to open an account on AWS, or to explain how to navigate the AWS console. 

# Can you go with a ready-made AMI?

Amazon Machine Images are the templates used to launch EC2 instances (the machine in the cloud we will be using). Louis Aslett has [built](https://www.louisaslett.com/RStudio_AMI/) a Rstudio server AMI, but as far as I can tell it is a bit outdated running on `Ubuntu 18.04` and `R 4.0.2`. I think this is perfectly fine for a lot of use cases. Unfortunately, the package I am developing uses the `|>` instead of the `%>%` pipe, and I had to make updates to make it work. Then I ran into some issues about keys being outdated and repositories not being enabled, and I decided I rather start from scratch instead of debugging the AMI.


# Step 1: Launch a new instance with Ubuntu 22.04 and install all R related packages

Launching a new instance with `Ubuntu 22.04` is a few clicks in which two things are important for later steps: a security group that will allow SSH traffic and HTTP traffic, and a key to log into the instance. Luckily if you forget to do any of these, the instance can be terminated and it is simple to start over.

Once the instance is launched we head over to our SSH terminal, log in, and install what is needed. Two handy references can be found on [Posit's website](https://posit.co/download/rstudio-server/) for `Rstudio server` and on [CRAN's website](https://cran.rstudio.com/bin/linux/ubuntu/) for `Ubuntu` packages for `R`.

Once everything is installed, and it a pretty quick installation process, `Rstudio server` is started automatically and you can see a notification about that in the SSH console.

The cool thing about Ubuntu is that almost all packages that I needed are available on their repositories, so I needed to do zero `install.packages()`  to compile packages from source. That is great.

# Step 2: Add a user for Rstudio and configure Rstudio server

It's a good practice to limit who can log in to Rstudio, and the Ubuntu forums answer this specific question on [how to add a new user](https://askubuntu.com/questions/838443/create-a-username-and-password-in-rstudio-server) on the system. Then follow the link to Posit's documentation about [Restricting by group)(https://docs.posit.co/ide/server-pro/authenticating_users/restricting_access.html).

Once a user is added and a password is set, you can also [configure](https://support.posit.co/hc/en-us/articles/200552316-Configuring-RStudio-Workbench-RStudio-Server) `Rsudio server` to run on port 80, and restart the `rstudio-server-service` with `sudu systemctl restart rstudio-server.service`. Assuming everything is correct the Rstudio login screen will show up on the public IP address of the EC2 instance.

Not need but useful addition to the configuration is to change the default shell for the newly created user by running  `chsh` in the SSH console. I prefer `bash`.

# Step 3: Set up a new SSH key for accessing github (or don't)

I needed this because I wanted to be able to pull from the Github repository where development is happening and to be able to potentially push any changes that will be made while working on the EC2. There is a handy how-to for this as well written by a github user [here](https://gist.github.com/aprilmintacpineda/f101bf5fd34f1e6664497cf4b9b9345f).

# Step 4: Set up an SSH tunnel so that the access to the Rstudio server is private

Instead of going through the public internet set up a tunnel with SSH - a cool explanation about tunneling is available in this [video](https://www.youtube.com/watch?v=AtuAdk4MwWw). First, remove the port 80 setting from the config file and restart the server. Then run `ssh -i yourkey.pem -L 8080:localhost:8787 ubuntu@ec2.your-instance.amazonaws.com` and `Rstudio server` becomes available on `localhost:8080`. Nice!

Now you are ready to do your amazing work in R using a machine with more ram or faster processor. :)

# Step 5: Don't forget to stop the instance when not working

This something to remember so the AWS bill doesn't accumulate costs, but also remember that the EBS volume will incur some costs on a stopped instance as well. A good practice is to set up budget alarms on AWS for keeping an eye on costs. 
---
title: Deploying Plumber API to AWS Elastic Container Service
author: "novica"
date: "2023-12-28"
slug: deploying-plumber-api-to-aws-elastic-container-service
categories:
  - AWS
tags:
  - AWS ECR
  - plumber
  - Connect
subtitle: ""
summary: "Notes on deploying Plumber API to AWS ECS."
authors: [novica]
lastmod: "2023-12-28T18:28:00+01:00"
featured: no
image:
  caption: ""
  focal_point: ""
  preview_only: no
projects: []
---

## Plumber API on ECS: Background

Recently, I started a new position where part of my role is to support colleagues
in the final steps of getting a data product out of the door. This will,
eventually, involve doing various administrative and development tasks on
[Posit Team](https://posit.co/products/enterprise/team/). However, before we
can actually get on the Posit Team train, a deadline was looming for deploying
a Plumber API. The API runs some simulations about host-parasite interactions
in marine fish and is a back-end for a video game that lets users modify parameters,
like treatments, to better understand parasite loads and fish biomass. This needed
to be tested before making it public.

Getting to the point where even one can thing about AWS was no simple task
though. One thing that almost always stands out when seeing `R` code in the
wild is that a lot of things still work in a non-reproducible way. Often with
`source`-ing scripts with hard-coded paths, and setting working directory
(`setwd()`) for setting the correct environment. It seems like `R` makes it
really easy to write code that works, but not necessarily in a production setting.

Of course, this is not novel insight, and is not meant to pile on criticism on
`R` users. Many of them are scientists, statisticians, or researchers whose main
priorities are doing science, writing mathematical models, and performing
advanced statistical analyses. This is not exactly the same as writing easily deployable 
code, which requires an altogether different experience and skillset, like reading source
code and software documentation, rather than research articles and experimental
protocols.

## Always write a package

I started where I usually start when refactoring a project: by converting the
scripts into an `R` package. The benefits of this are documented in
[many](https://r-pkgs.org/) [places](https://kbroman.org/pkg_primer/pages/why.html)
[across](https://www.jumpingrivers.com/blog/personal-r-package/) the internet,
and specifically the topic for [adding](https://community.rstudio.com/t/plumber-api-and-package-structure/18099)
an API to a [package](https://www.harveyl888.com/post/2022-11-11_plumber_as_package/) has popped up a few times as well.

A common pattern for packaging `{plumber}` code is to write and export functions
in the same way one would for a typical `R` package. Then, after
loading the package, the API endpoints simply call exported functions. The API
typically lives in `inst`, so its available as a `system.file` after installing
the package, which is important, and simplifies things with Docker
downstream.

All of these considerations were important for this project, since further down
the line, people with no particular experience in `R` needed to
test the API (in our case the game developers). And once the package was there 
it was simple to create a `docker` image that installs the package, 
and runs the API with the help of the [dockerfiler](https://github.com/ThinkR-open/dockerfiler) 
package.

## AWS ECS

However, for a real test of the API we needed to have it public on the internet.
Having the `docker` image ready, I was happy to learn that it is relatively simple to
deploy containerized applications on AWS using [AWS Copilot-cli](https://aws.github.io/copilot-cli/docs/getting-started/first-app-tutorial/).

There are some pre-requisites to deploying wit `copilot-cli`. Of course you need 
an AWS account. And you should set up your AWS credentials, such as default region
and `access key`. The easiest way to go about this is to use the `aws cli` to set 
it up. After [installing](https://aws.amazon.com/cli/) the `cli` for your operating 
system, running `aws configure` should prompt you to enter the needed credentials. 
After you set that up, `aws configure list` should print the current profile, 
and you can check that everything is correct.

The next step is to install `copilot-cli` for your [operating system](https://aws.github.io/copilot-cli/docs/getting-started/install/). The 
documentation for `copilot-cli` provides an example app to be deployed. It is a 
good idea to deploy the [example](https://aws.github.io/copilot-cli/docs/getting-started/first-app-tutorial/) just see what you can expect when deploying your container as well.

If you browse through the example repository after the deployment is completed 
you will see some of the new files that were created by `copilot`. The `example-app`
folder has a file called `manifest.yml` which has the details for the app as 
infrastructure-as-code. Going through this file is useful to understand what is
happening behind the scenes.

Deploying the API then was a simple process. Running `copilot init` in the API repository
prompts to answer few questions -- the same as in the example. 
Then `copilot` sets up the infrastructure and deploys the test environment. 

Success! Or so I thought. The first deployment of the API failed.

There were couple of tweaks that I needed to do. 

One, I needed to set up a custom
health check path. The "Hello World" endpoint of the API was at  '/hello', so I 
had to correct that in the `manifest.yml` file. 

Two, I needed this API to run over HTTPS, so I had to add [domain settings](https://aws.github.io/copilot-cli/docs/manifest/lb-web-service/) to 
the Load Balanced Web Service. I was lucky enough to have one of those
domains-for-a-side-project-I-never-got-to-do lying around, so at least I didn't
have to buy a new domain for this test deployment.

Three, I added Autoscaling to the service, and increased the CPU and Memory size 
used by the containers. This was a good way to see how the
API will handle the tests. All of these changes go into the `manifest.yml` file. 

## Clean up

After the testing was done, removing everything from AWS was a simple thing.
Running `copilot app delete` deletes all the resources that `copilot` set up for
the API. This is great because you don't have to worry about forgetting some 
resource which will continue to incur costs on AWS.

## Summary

This article provides an overview of deploying an R Plumber API in as a 
containerized application on AWS using `copilot-cli`.
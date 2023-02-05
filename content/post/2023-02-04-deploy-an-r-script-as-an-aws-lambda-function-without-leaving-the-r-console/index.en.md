---
title: Deploy an R script as an AWS Lambda function without leaving the R console
author: Teo
date: '2023-02-04'
slug: deploy-an-r-script-as-an-aws-lambda-function-without-leaving-the-r-console
categories:
  - R
  - AWS
tags:
  - paws
  - AWS Lambda
  - AWS IAM
  - AWS ECR
  - Docker
  - package
subtitle: ''
summary: ''
authors: ['teo']
lastmod: '2023-02-04T22:02:01-06:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: ['r2lambda']
---

# Overview and motivation

AWS Lambda is a serverless compute service that lets us deploy any code as a cloud
function without worrying about setting up a compute server (for example like discussed
[here](/post/rstudio-in-the-cloud-for-those-of-us-with-old-laptops/) 
and [here](/post/rstudio-in-the-cloud-for-those-of-us-with-old-laptops-part-2-automating-with-terraform/)). 
The code can be deployed as a zip archive or a container image. In the `R` case, 
only the container option is available and enabled by the [`{lambdr}`](https://lambdr.mdneuzerling.com/index.html) 
package (see also this [workflow](https://github.com/Appsilon/r-lambda-workflow)). 
The deployment using `{lambdr}`procedure involves:  

  1) writing an `R` script to serve as the runtime function  
  2) creating a Lambda Dockerfile and docker image locally  
  3) pushing the docker image to AWS Elastic Container Registry (ECR)  
  4) creating a Lambda function using the ECR image either with the AWS 
  command-line interface (`cli`) or in the web console     

Going through this procedure is a rewarding experience, as it teaches many things
related to docker containers, AWS setup, using the AWS `cli`, etc. But,
once you have gone through the this procedure a few times, it becomes a bit 
cumbersome and time-consuming to navigate between an `R` console, the `shell`, and
`aws cli` or the AWS web console in a browser to write and update the code, 
re-create the docker image, push, and then re-create the Lambda function. It would be 
nice to have a "one-call" solution, where we call a single function, point it to a
file that contains the code we wish to deploy, and sit back while our `R` session
goes through the steps.

In this post, I will introduce an `R` package, called `{r2lambda}`, that aims to make 
it easier to deploy `R` code as AWS Lambda function by automating the above procedure. 

# `{r2lambda}`

Once everything is set up (see below), `{r2lambda}` will let you do the following:

1. Write your `R` code:

``` r
library("x")
library("y")

my_fun <- function(arg1) {
  # stuff happening to `arg`
}

lambdr::start_lambda()
```

and save to a file (e.g., 'my_lambda.R').

2. Call `r2lambda::deploy_lambda()` to create the AWS Lambda function in one line of code:

``` r
deploy_lambda(
  tag = "my-lambda",
  runtime_function = "my_fun",
  runtime_path = "path/to/script/my_lambda.R",
  dependencies = c("x", "y")
  )
```

Where, 

- `tag` becomes the name of the docker image and lambda function  
- `runtime_function` is the function we want the docker/lambda to run when invoked
- `runtime_path` is the path to the script 
- `dependencies` is a character vector of dependencies to install in the docker image 

This step usually takes a few minutes, because we are pushing a potentially large `docker`
image to the cloud.

3. Test your Lambda using `r2lambda::invoke_lambda()`:

``` r
invoke_lambda(
  function_name = "my-lambda",
  payload = list(arg1 = 1),
  invocation_type = "RequestResponse"
 )
```

Where, 

- `function_name` is the same as the `tag` argument of `deploy_lambda`
- `payload` is a named list of arguments that the runtime function `my_fun` takes
- `invocation_type` is the type of invocation (can also be `DryRun` and `Event`)

The named list in the payload is converted to `json` internally before sending the request.

That's it! That is all you need to do to deploy a basic Lambda function from 
your `R` script using `{r2lambda}`. I emphasized 'basic', because we don't yet have
ways to customize the lambda environment (API gateway, events, granting access to 
other services, etc). But some of this functionality is already planned and hopefully
coming soon.

# Setup

With the main usage out of the way, lets go over some of the implementation
details, environment setup, and installation.

## System dependencies

The only system dependency is `docker`, because `R` lambdas are `docker` images, 
so we are not going anywhere unless docker is [installed](https://docs.docker.com/get-docker/). 

## `R` dependencies

The core `R` dependencies are [`{lambdr}`](https://github.com/mdneuzerling/lambdr/), 
[`{stevedore}`](https://github.com/richfitz/stevedore), and [`{paws}`](https://paws-r.github.io/). 

`{lambdr}` provides the `R` runtime for AWS Lambda. In practice, the most important 
points about using `{lambdr}` are 1) to setup the Dockerfile:  

  - installation of system dependencies   
  - installation of R dependencies of the runtime function  
  - setting the runtime function to be run by the container via `CMD`,  
      
and 2) to setup the `R` script by adding `lambdr::start_lambda()` at the bottom. 

`{stevedore}` is a `docker` client for `R`. It provides an interface to the `docker` API.
In the context of deploiyng Lambda functions, it is used to list and tag local images,
and to login and push images to the AWS ECR repository. Using `{stevedore}` simplifies
how `{r2lambda}` works in two ways. First, we don't need to use system calls to run
`{docker}` commands, and second, we don't depend on the `aws cli`.

`{paws}` is the `R` software development kit for AWS. `{r2lambda}` uses `{paws}` 
to connect to the AWS services using your credentials, to create execution roles 
for the Lambda, to create the Lambda it self, and to invoke the Lambda function.

The remaining `R` dependencies provide features for input validation and testing
(`{checkmate}`), logging (`{logger}`), and text interpolation (`{glue}`). 

## Environmental variables for AWS configuration  

Our AWS credentials are needed so that functions that use the `paws` SDK can 
authenticate with `AWS`. This is a simple `.Renviron`:

``` r
ACCESS_KEY_ID = "YOUR AWS ACCESS KEY ID"
SECRET_ACCESS_KEY = "YOUR AWS SECRET ACCESS KEY"
PROFILE = "YOUR AWS PROFILE"
REGION = "YOUR AWS REGION"
```

## Installation

Once the prerequisites are ready, we can install `{r2lambda}` from [`GitHub`](https://github.com/) 
using `{remotes}`:

``` r
# install_packages("remotes")
remotes::install_github("discindo", "r2lambda")
```

## Demo run with logs

A full deployment and invocation run with a demo runtime script should look like 
the code and output below. 

``` r
>   runtime_function <- "parity"
>   runtime_path <- system.file("parity.R", package = "r2lambda")
>   dependencies <- NULL
> 
>   deploy_lambda(
+     tag = "parity-test1",
+     runtime_function = runtime_function,
+     runtime_path = runtime_path,
+     dependencies = dependencies
+     )
INFO [2023-01-29 20:32:41] [deploy_lambda] Checking system dependencies (`aws cli`, `docker`).
/usr/bin/docker
INFO [2023-01-29 20:32:41] [deploy_lambda] Creating temporary working directory.
INFO [2023-01-29 20:32:41] [deploy_lambda] Creating Dockerfile.
WARN [2023-01-29 20:32:41] [deploy_lambda] Created Dockerfile and lambda runtime script in temporary folder.
INFO [2023-01-29 20:32:41] [deploy_lambda] Building Docker image.
Sending build context to Docker daemon  3.584kB
Step 1/13 : FROM public.ecr.aws/lambda/provided
 ---> ccae8d728af2
Step 2/13 : ENV R_VERSION=4.0.3
 ---> Using cache
 ---> bf3dd3c804f3
Step 3/13 : RUN yum -y install wget git tar
 ---> Using cache
 ---> 8b82b80771cf
Step 4/13 : RUN yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm   && wget https://cdn.rstudio.com/r/centos-7/pkgs/R-${R_VERSION}-1-1.x86_64.rpm   && yum -y install R-${R_VERSION}-1-1.x86_64.rpm   && rm R-${R_VERSION}-1-1.x86_64.rpm
 ---> Using cache
 ---> c98bc560eff4
Step 5/13 : ENV PATH="${PATH}:/opt/R/${R_VERSION}/bin/"
 ---> Using cache
 ---> 4565b8100c39
Step 6/13 : RUN yum -y install openssl-devel
 ---> Using cache
 ---> d8e46fe52a6d
Step 7/13 : RUN Rscript -e "install.packages(c('httr', 'jsonlite', 'logger', 'remotes'), repos = 'https://packagemanager.rstudio.com/all/__linux__/centos7/latest')"
 ---> Using cache
 ---> 46ca4b6e95a0
Step 8/13 : RUN Rscript -e "remotes::install_github('mdneuzerling/lambdr')"
 ---> Using cache
 ---> 67283a940985
Step 9/13 : RUN mkdir /lambda
 ---> Using cache
 ---> d6762390f9a9
Step 10/13 : COPY runtime.R /lambda
 ---> Using cache
 ---> 94af1e345ecc
Step 11/13 : RUN chmod 755 -R /lambda
 ---> Using cache
 ---> cd15870ad843
Step 12/13 : RUN printf '#!/bin/sh\ncd /lambda\nRscript runtime.R' > /var/runtime/bootstrap   && chmod +x /var/runtime/bootstrap
 ---> Using cache
 ---> 66d74d4de62e
Step 13/13 : CMD ["parity"]
 ---> Using cache
 ---> e47d4fea17a1
Successfully built e47d4fea17a1
Successfully tagged parity-test1:latest
WARN [2023-01-29 20:32:41] [deploy_lambda] Docker image built. This can take up substantial amount of disk space.
WARN [2023-01-29 20:32:41] [deploy_lambda] Use `docker image ls` in your shell to see the image size.
WARN [2023-01-29 20:32:41] [deploy_lambda] Use `docker rmi <image>` in your shell to remove an image.
INFO [2023-01-29 20:32:41] [deploy_lambda] Pushing Docker image to AWS ECR.

... [truncated]

Login Succeeded
The push refers to repository [*.dkr.ecr.us-east-1.amazonaws.com/parity-test1]

... [truncated]

latest: digest: sha256:9f38150cf89bf6a3f7d95c853105afe82616f85f3afbb65d7f71d2f1400dedeb size: 3045
WARN [2023-01-29 20:45:25] [deploy_lambda] Docker image pushed to ECR. This can take up substantial resources and incur cost.
WARN [2023-01-29 20:45:25] [deploy_lambda] Use `paws::ecr()`, the AWS CLI, or the AWS console to manage your images.
INFO [2023-01-29 20:45:25] [deploy_lambda] Creating Lambda role and basic policy.
WARN [2023-01-29 20:45:26] [deploy_lambda] Created AWS role with basic lambda execution permissions.
WARN [2023-01-29 20:45:26] [deploy_lambda] Use `paws::iam()`, the AWS CLI, or the AWS console to manage your roles, and permissions.
INFO [2023-01-29 20:45:36] [deploy_lambda] Creating Lambda function from image.
WARN [2023-01-29 20:45:37] [deploy_lambda] Lambda function created. This can take up substantial resources and incur cost.
WARN [2023-01-29 20:45:37] [deploy_lambda] Use `paws::lambda()`, the AWS CLI, or the AWS console to manage your functions.
WARN [2023-01-29 20:45:37] [deploy_lambda] Lambda function created successfully.
WARN [2023-01-29 20:45:37] [deploy_lambda] Pushed docker image to ECR with URI `*.dkr.ecr.us-east-1.amazonaws.com/parity-test1`
WARN [2023-01-29 20:45:37] [deploy_lambda] Created Lambda execution role with ARN `arn:aws:iam::*:role/parity-test1--261b7f62-a048-11ed-bd89-10c37b6dce99`
WARN [2023-01-29 20:45:37] [deploy_lambda] Created Lambda function `parity-test1` with ARN `arn:aws:lambda:us-east-1:*:function:parity-test1`
SUCCESS [2023-01-29 20:45:37] [deploy_lambda] Done.
> 
>  invoke_lambda(
+    function_name = "parity-test1",
+    payload = list(number = 3),
+    invocation_type = "RequestResponse"
+   )
INFO [2023-01-29 20:45:37] [invoke_lambda] Validating inputs.
INFO [2023-01-29 20:45:37] [invoke_lambda] Invoking function.
Error: ResourceConflictException (HTTP 409). The operation cannot be performed at this time. The function is currently in the following state: Pending

>  invoke_lambda(
+    function_name = "parity-test1",
+    payload = list(number = 3),
+    invocation_type = "RequestResponse"
+   )
INFO [2023-01-29 21:32:53] [invoke_lambda] Validating inputs.
INFO [2023-01-29 21:32:53] [invoke_lambda] Invoking function.

Lambda response payload: 
{"parity":"odd"}
SUCCESS [2023-01-29 21:33:02] [invoke_lambda] Done.
```

The code is purposely verbose to let 
the user know what actions are being taken and what resources are being set up on
AWS. Of course, setting up and running services on AWS incurs costs, so it is
always a good idea to review your AWS console and disable or remove services that 
are no longer needed. Actions like these can also be done with the `paws` SDK, and 
I hope that a future iteration of `{r2lamdba}` might make some AWS clean up possible 
from the `R` console. For now, `deploy_lambda` will specify the URI or ARN of each 
service it creates both as a console log, and in the returned list. So the user 
can easily find the created services and disable/remove/update as needed. As of now,
`{r2lambda}` interacts only with IAM, ECR, and Lambda, so be sure at least to log 
into AWS, and review the actions taken.

# TODO list

`{r2lambda}` is a not more than a week old, work-in-progress package. It fulfills a relatively simple 
version of the initial goal --one-function deploy of an `R` script with CRAN dependencies-- 
but it is not quite ready for wider usage. I think several important features would
add much more value to the user, including:

- managing dependencies from different repositories. As of now, only CRAN packages are
supported. The `deps` argument is passed onto a simple function that adds an `install.packages`
call to the Dockerfile, so any package hosted in a git repository or on Bioconductor,
would not be able to be installed. 

- detecting dependencies via `{attachment}` or `{dockerfiler}` and populating the 
Dockerfile with the correct installation calls.

- adding functions that would allow removing of services from AWS. For example, if
creating a Lambda function is a one function call, it would be quite useful if 
cleaning up after an erroneous deployment is a one function call as well. This would include
removing the Lambda function, any IAM roles and attached policies to the Lambda, as 
well as the ECR image associated with the function.

- setting up a way to test the Lambda function locally _before_ deploying to AWS. This is
described [here](https://docs.aws.amazon.com/lambda/latest/dg/images-test.html) and 
is something I do regularly to avoid premature pushing to ECR, as this is the most 
time-consuming step. But I am yet to add this routine to the package.

- making it easier to customize the Lambda environment on deploy. To support at least
basic configurations like memory and timeout limits but also adding additional policies
(e.g., permissions to put objects in AWS S3 or interact with a database service like 
RDS or DynamoDB). 

# Summary

Please give `{r2lambda}` a try and share your feedback by commenting in the 
repository [Discussions](https://github.com/discindo/r2lambda/discussions). Would 
love to hear about your experience, any problems you might have encountered,
any similar tools you might know about, and of courses, improvement ideas. 

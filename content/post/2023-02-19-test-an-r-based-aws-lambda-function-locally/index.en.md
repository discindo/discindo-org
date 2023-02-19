---
title: Test an R-based AWS lambda function locally before deploying to the cloud
author: teo
date: '2023-02-19'
slug: test-an-r-based-aws-lambda-function-locally
categories:
  - AWS
  - R
tags:
  - paws
  - AWS Lambda
  - AWS IAM
  - AWS ECR
  - Docker
  - package
subtitle: ''
summary: ''
authors: [teo]
lastmod: '2023-02-19T14:44:41-06:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: ['r2lambda']
show_related: true
---

Creating AWS Lambda functions from `R` code can be a powerful way to make our 
local `R` code available in the cloud as an on-demand serverless service. In recent 
weeks I've been working on my package [r2lambda](https://github.com/discindo/r2lambda) 
to build and deploy AWS Lambda functions from the `R` console. In an introductory [article](https://discindo.org/post/deploy-an-r-script-as-an-aws-lambda-function-without-leaving-the-r-console/) 
about the package a few weeks ago, I covered the basics and showcased the
main usage:

``` r

# install_packages("remotes")
remotes::install_github("discindo/r2lambda")

r2lambda::deploy_lambda(
  tag = "my-lambda",
  runtime_function = "my_fun",
  runtime_path = "path/to/script/of/my_fun",
  dependencies = c("ggplot2", "dplyr")
  )
  
```

In my first post on the topic, I noted that this very much a work in progress, and
that I hope to actively develop this package by adding features that would make it 
useful in more realistic scenarios. 

# Testing R-based AWS Lambda functions locally

One such feature was the ability to test our `R`-based Lambda function locally 
before deploying to the AWS cloud. This is super useful, because, depending on 
the size of the `docker` image it might take a while to push it to the 
AWS ECR repository. Also, creating an AWS Lambda function from the ECR docker image,
requires granting the function a role and permissions policy to execute or access
other services. All of these steps create resources/services in your AWS account, 
so ideally, would only be done when we are certain that docker image we are deploying
as a Lambda function works correctly.

This procedure is well [documented](https://docs.aws.amazon.com/lambda/latest/dg/images-test.html).
After creating the Lambda `docker` image, to test it locally, we need to 1) run 
a container on our local machine and 2) send a request to it with `curl`. Essentially,
we are 'invoking' the function with the same payload the same way we'd do it in the cloud,
but locally. This is the best way to know that everything works as intended. Merely
testing the `R` code separately, without the Lambda `docker` context, might not be 
enough.

I packaged this routine in the function `test_lambda` and added this step to the
`{r2lamdba}` deployment workflow. With these changes, instead of one function
`deploy_lambda` that would build and deploy the image, we now have thee steps:

1. `build_lambda` -- to build and tag the `docker` image locally
2. `test_lambda` -- to test the lambda docker container locally (optional but recommended)
3. `deploy_lambda` -- to push the docker image to the cloud and create the function

Or in code:

### Build a docker image for the lambda function

``` r
runtime_function <- "parity"
runtime_path <- system.file("parity.R", package = "r2lambda")
dependencies <- NULL

# Might take a while, its building a docker image
build_lambda(
 tag = "parity1",
 runtime_function = runtime_function,
 runtime_path = runtime_path,
 dependencies = dependencies
 )
```

### Test the lambda docker image locally

``` r
payload <- list(number = 2)
tag <- "parity1"
test_lambda(tag = "parity1", payload)
```

### Deploy to AWS Lambda

``` r
# Might take a while, its pushing it to a remote repository
deploy_lambda(tag = "parity1")
```

### Invoke deployed lambda

``` r
invoke_lambda(
  function_name = "parity1",
  invocation_type = "RequestResponse",
  payload = list(number = 2),
  include_logs = FALSE
)

#> Lambda response payload: 
#> {"parity":"even"}
```

So, although we've added a couple of steps to the workflow, I think its for the 
better, as we can have finer control over building and deploying. For example,
sometimes we might want to deploy a Lambda function from an existing `docker` image,
so de-coupling the build and deploy steps makes a lot of sense. 

Would love to hear from you! Let me know if you try the `r2lamdba` package or if you
know of any similar projects. 


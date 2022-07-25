---
title: 'How to invoke an AWS Lambda function with R and paws'
author: Teo
date: '2022-07-24'
slug: invoke-an-aws-lambda-function-with-r-and-paws
categories:
  - R
tags:
  - aws
  - paws
  - AWS lambda
  - .Renviron
subtitle: ''
summary: ''
authors: ['teo']
lastmod: '2022-07-24T16:42:34-06:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

In recent weeks, I have been trying to learn more about using the `{paws}` software development kit to
interact with Amazon Web Services from `R`. So far, my focus was on the basics of `DynamoDB`. How
to put one item in the database, and how to migrate reasonably sized table into `DynamoDB`. There are 
more topics on `DynamoDB` that I would focus on, and I hope to document my experience in future posts.
But today, I wanted to switch gears to `AWS Lambda`. Specifically, how to invoke an already deployed 
cloud function from `R`. 

## AWS Setup

In my `R`+`AWS` projects I typically include an `.Renviron` file that stores the 
needed settings and secrets for authenticating with AWS. These environmental variables
are passed as config when starting services with `paws`. A minimal `.Renviron` for 
this purpose might look like so:

```{r}
ACCESS_KEY_ID = "MYKEY"
SECRET_ACCESS_KEY = "MYSECRET"
PROFILE = "default"
REGION = "us-east-1"
```

Then, within our `R` session, we would get these configurations with the typical:

```{r}
Sys.getenv("REGION")
Sys.getenv("PROFILE")
```

Alternatively, if we can't or don't want to store secrets in a file, we could set 
them directly in the code:

```{r}
Sys.setenv(REGION = 'us-east-2')
```

## Starting an AWS `Lambda` service in `R`

Now, with our configuration prerequisites in place, we can establish a local lambda 
service using `paws::lambda`. 

```{r}
lambda_service <- paws::lambda(config = list(
  credentials = list(
    creds = list(
      access_key_id = Sys.getenv("ACCESS_KEY_ID"),
      secret_access_key = Sys.getenv("SECRET_ACCESS_KEY")
    ),
    profile = Sys.getenv("PROFILE")
  ),
  region = Sys.getenv("REGION")
))
```

With the service started, we can see the functions available in our AWS cloud, and the
operations we can perform.

```{r}
> all_lambdas <- lambda_service$list_functions()
> sapply(all_lambdas$Functions, "[[" , "FunctionName")
[1] "diamonds"     "parity"       "r-lambda-poc"
```

Currently, I have three deployed `Lambda` functions, two of which are examples from 
David Neuzerling's excellent work with the [{lambdr}](https://lambdr.mdneuzerling.com/) 
`R` package. For more details on these (diamonds and parity), see David's writing on the topic [here](https://mdneuzerling.com/post/serverless-on-demand-parametrised-r-markdown-reports-with-aws-lambda/),
[here](https://mdneuzerling.com/post/r-on-aws-lambda-with-containers/), and 
[here](https://lambdr.mdneuzerling.com/articles/lambda-runtime-in-container.html), 
which was and still is incredibly helpful for me to get started and keep learning.

## Invoking an `AWS lambda` function with `R` and `{paws}`

As an example, I am going to use the `parity` lambda, described in detail in 
one of the [vignettes](https://lambdr.mdneuzerling.com/articles/lambda-runtime-in-container.html) 
of the `{lambdr}` package. The input (payload) of this lambda function is a named list of the form 
`list(number = 2)`, which in `JSON` becomes:

```{json}
# parity lambda input payload
{"number": 2}
```

The return is also a one-item named list with `JSON` `{"parity": "odd"|"even"}`. The `R` function
run to assess the parity is the following:

```{r}
parity <- function(number) {
  list(parity = if (as.integer(number) %% 2 == 0) "even" else "odd")
}
```

To invoke this function we need to call the `invoke` operation of our lambda service,
and supply it with 1) the name of the function (or function `arn`), 2) the type of 
invocation ('DryRun', 'RequestResponse', or 'Event' see `?paws.compute::lambda_invoke`), 
and 3) the input payload  (in this case `'{ "number": "2" }'`). We'll also ask for 
the tail of the execution log so we can get more information regarding the duration, 
memory usage, and billing information. 

```{r}
# invoke the parity lambda
response <- lambda_service$invoke(
  FunctionName = "parity",
  InvocationType = "RequestResponse",
  Payload = '{ "number": "2" }',
  LogType = "Tail"
)
```

After a second or two, we can decode the output payload, we are expecting `parity = "even"`:

```{r}
> response$Payload |> rawToChar() |> cat()
{"parity":"even"}
```

For the logs, we should first decode the response string, as it comes `base64` encoded. And then,
convert to a human-readable character vector:

```{r}
> jsonlite::base64_dec(response$LogResult) |> rawToChar() |> cat()
START RequestId: 7c971dd1-938d-4ae7-ad35-e1ee5b59e6bb Version: $LATEST
END RequestId: 7c971dd1-938d-4ae7-ad35-e1ee5b59e6bb
REPORT RequestId: 7c971dd1-938d-4ae7-ad35-e1ee5b59e6bb	Duration: 778.92 ms	Billed Duration: 3002 ms	Memory Size: 128 MB	Max Memory Used: 98 MB	Init Duration: 2222.94 ms
```

(Using `cat()` here because it handles newlines nicely)

## Using `paws` bulding blocks to build a custom AWS `Lambda` invocation function

First, we can wrap the call to `paws::lambda` in a function so we can use it to connect
to different AWS services. For now, we assume the environmental variables are set, but
we would implement checks and error handling in production setting.

```{r}
aws_connect <- function(service) {
  service(config = list(
    credentials = list(
      creds = list(
        access_key_id = Sys.getenv("ACCESS_KEY_ID"),
        secret_access_key = Sys.getenv("SECRET_ACCESS_KEY")
      ),
      profile = Sys.getenv("PROFILE")
    ),
    region = Sys.getenv("REGION")
  ))
}
```

With this function, we can now quickly create a `dynamoDB` and `Lambda` services:

```{r}
lambda_service <- aws_connect(paws::lambda)
lambda_service$list_functions()
# output truncated

dynamodb_service <- aws_connect(paws::dynamodb)
dynamodb_service$list_tables()
# output truncated
```

Then, we go to the next step, to wrap the service start and invocation in one function. The
payload is a named list, which internally is converted to `JSON` by `jsonlite::toJSON`.

```{r}
invoke_lambda <-
  function(function_name,
           invocation_type,
           payload,
           include_logs = FALSE) {
    # assumes .Renviron is set up
    lambda_service <- aws_connect(paws::lambda)
    
    response <- lambda_service$invoke(
      FunctionName = function_name,
      InvocationType = invocation_type,
      Payload = jsonlite::toJSON(payload),
      LogType = ifelse(include_logs, "Tail", "None")
    )
    
    message("\nLambda response payload: ")
    response$Payload |> rawToChar() |> cat()
    
    if (include_logs) {
      message("\nLambda logs: ")
      jsonlite::base64_dec(response$LogResult) |> rawToChar() |> cat()
    }
    
    invisible(response)
  }

```

Finally, we run our simple helpful wrapper:

```{r}
> invoke_lambda(
+   function_name = "parity",
+   invocation_type = "RequestResponse",
+   payload = list(number = 5),
+   include_logs = FALSE
+ )

Lambda response payload: 
{"parity":"odd"}

> invoke_lambda(
+   function_name = "parity",
+   invocation_type = "RequestResponse",
+   payload = list(number = 5),
+   include_logs = TRUE
+ )

Lambda response payload: 
{"parity":"odd"}
Lambda logs: 
START RequestId: b3507d9d-1218-4c1a-8a2a-d27c728b5093 Version: $LATEST
END RequestId: b3507d9d-1218-4c1a-8a2a-d27c728b5093
REPORT RequestId: b3507d9d-1218-4c1a-8a2a-d27c728b5093	Duration: 134.67 ms	Billed Duration: 135 ms	Memory Size: 128 MB	Max Memory Used: 101 MB

```

## Summary

These were some baby steps in accessing AWS `Lambda` services and invoking functions 
from `R` using `paws`. There is a lot more to be done with `paws` and `lambda`, 
including creating lambda functions, linking them to other AWS services, using them
in `{shiny}` applications, etc. I hope to cover some of these in future posts.
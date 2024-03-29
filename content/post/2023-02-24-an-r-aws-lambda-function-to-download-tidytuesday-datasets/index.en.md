---
title: An R AWS Lambda function to download Tidytuesday datasets
author: 'teo'
date: '2023-02-24'
slug: an-r-aws-lambda-function-to-download-tidytuesday-datasets
categories:
  - AWS
  - R
tags:
  - r2lambda
  - AWS Lambda
  - tidytuesday
subtitle: ''
summary: 'A detailed walk through the steps to prepare a custom R script for deployment as AWS Lambda with the `r2lambda` package. How to prepare an `R` script? What are the roles of several key arguments? How to request longer timeout or more memory for a Lambda function? How to parse the response payload returned by the Lambda function?'
authors: [teo]
lastmod: '2023-02-24T23:37:01-06:00'
featured: yes
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: ['r2lambda']
show_related: true
---

## Use `{r2lambda}` to download Tidytuesday dataset

In this exercise, we'll create an AWS Lambda function that downloads
the [tidytuesday](https://github.com/rfordatascience/tidytuesday/tree/master/data/2023/2023-02-07) 
data set for the most recent Tuesday (or most recent Tuesday from a date of interest).

## Required packages

```{r setup}
library(r2lambda)
library(jsonlite)
library(magrittr)
```

## Runtime function

The first step is to write the runtime function. This is the function that will be
executed when we invoke the Lambda function after it has been deployed. To download 
the Tidytuesday data set, we will use the `{tidytuesdayR}` package. In the runtime 
script, we define a function called `tidytyesday_lambda` that takes one optional 
argument `date`. If `date` is omitted, the function returns the data set(s) for the most 
recent Tuesday, otherwise, it looks up the most recent Tuesday from a date of interest 
and returns the corresponding data set(s).

```{r}
library(tidytuesdayR)

tidytuesday_lambda <- function(date = NULL) {
  if (is.null(date))
    date <- Sys.Date()
  
  most_recent_tuesday <- tidytuesdayR::last_tuesday(date = date)
  tt_data <- tidytuesdayR::tt_load(x = most_recent_tuesday)
  data_names <- names(tt_data)
  data_list <- lapply(data_names, function(x) tt_data[[x]])
  return(data_list)
}

tidytuesday_lambda("2022-02-02")
```

## R script to build the lambda

To build the lambda image, we need an `R` script that sources any required code,
loads any needed libraries, defines a runtime function, and ends with a call to 
`lambdr::start_lambda()`. The runtime function does not have to be defined in this 
file. We could, for example, source another script, or load a package and set a 
loaded function as the runtime function in the subsequent call to `r2lambda::build_lambda`
(see below). We save this script to a file and record the path:

```{r}
r_code <- "
  library(tidytuesdayR)

  tidytuesday_lambda <- function(date = NULL) {
    if (is.null(date))
      date <- Sys.Date()
    
    most_recent_tuesday <- tidytuesdayR::last_tuesday(date = date)
    tt_data <- tidytuesdayR::tt_load(x = most_recent_tuesday)
    data_names <- names(tt_data)
    data_list <- lapply(data_names, function(x) tt_data[[x]])
    return(data_list)
  }
  
  lambdr::start_lambda()
"

tmpfile <- tempfile(pattern = "ttlambda_", fileext = ".R")
write(x = r_code, file = tmpfile)
```

## Build, test, and deploy the lambda function

### 1. Build

- We set the `runtime_function` argument to the name of the function we wish the 
`docker` container to run when invoked. In this case, this is `tidytuesday_lambda`.
This adds a `CMD` instruction to the `Dockerfile`

- We set the `runtime_path` argument to the path we stored the script defining our
runtime function.

- We set the `dependencies` argument to `c("tidytuesdayR")`because we need to 
have the `tidytuesdayR` package installed within the `docker` container if we are
to download the dataset. This steps adds a `RUN` instruction to the `Dockerfile`
that calls `install.packages` to install `{tidytuesdayR}` from CRAN.

- Finally, the `tag` argument sets the name of our Lambda function which we'll use 
later to test and invoke the function. The `tag` argument also becomes the name of 
the folder that `{r2lambda}` will create to build the image. This folder will have
two files, `Dockerfile` and `runtime.R`. `runtime.R` is our script from `runtime_path`,
renamed before it is copied in the `docker` image with a `COPY` instruction.

```{r}
runtime_function <- "tidytuesday_lambda"
runtime_path <- tmpfile
dependencies <- "tidytuesdayR"

r2lambda::build_lambda(
  tag = "tidytuesday3",
  runtime_function = runtime_function,
  runtime_path = runtime_path,
  dependencies = dependencies
)
```

### 2. Test

To make sure our Lambda `docker` container works as intended, we start it locally,
and invoke it to test the response. The response is a list of three elements:

```{r}
response <- r2lambda::test_lambda(tag = "tidytuesday3", payload = list(date = Sys.Date()))
```

- `status`, should be 0 if the test worked,
- `stdout`, the standard output stream of the invocation, and
- `stderr`, the standard error stream of the invocation

`stdout` and `stderr` are `raw` vectors that we need to parse, for example:

```{r}
rawToChar(response$stdout) 
```

If the `stdout` slot of the response returns the correct output of our function,
we are good to deploy to AWS.

### 3. Deploy

The deployment step is simple, in that all we need to do is specify the name (tag) of 
the Lambda function we wish to push to AWS ECR. The `deploy_lambda` function also
accepts `...`, which are named arguments ultimately passed onto 
`paws.compute:::lambda_create_function`. This is the function that calls the Lambda
API. To see all available arguments run `?paws.compute:::lambda_create_function`.

The most important arguments are probably `Timeout` and `MemorySize`, which set 
the time our function will be allowed to run and the amount of memory it will have
available. In many cases it will make sense to increase the defaults of 3 seconds
and 128 mb.

```{r}
r2lambda::deploy_lambda(tag = "tidytuesday3", Timeout = 30)
```

### 4. Invoke 

If all goes well, our function should now be available on the cloud awaiting requests.
We can invoke it from `R` using `invoke_lambda`. The arguments are:

- `function_name` -- the name of the function
- `invocation_type` -- typically `RequestResponse`
- `include_log` -- whether to print the logs of the run on the console
- `payload` -- a named list with arguments sent to the `runtime_function`. In this
case, the runtime function, `tidytuesday_lambda` has a single argument `date`, so
the corresponding list is `list(date = Sys.Date())`. As our function can be called
without any argument, we can also send an empty list as the payload.

```{r}
response <- r2lambda::invoke_lambda(
  function_name = "tidytuesday3",
  invocation_type = "RequestResponse",
  payload = list(),
  include_logs = TRUE
)
```

Just like in the local test, the response payload comes as a raw vector that needs to 
be parsed into a data.frame:

```{r}
tidytuesday_dataset <- response$Payload %>% 
  rawToChar() %>% 
  jsonlite::fromJSON(simplifyDataFrame = TRUE)

tidytuesday_dataset[[1]][1:5, 1:5]
```

## Summary

In this post, we went over some details about:

- how to prepare an `R` script before deploying it as a Lambda function,   
- what are the roles of several of the key arguments,   
- how to request longer timeout or more memory for a Lambda function, and    
- how to parse the response payload returned by the Lambda function

Stay tuned for a follow-up post where we set this Lambda function to run on a 
weekly schedule!

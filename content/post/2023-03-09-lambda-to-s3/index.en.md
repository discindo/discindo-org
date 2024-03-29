---
title: 'How to set up an R-based AWS Lambda to write to AWS S3 on a schedule'
author: 'teo'
date: '2023-03-09'
slug: lambda-to-s3
categories:
  - AWS
  - R
tags:
  - r2lambda
  - AWS Lambda
  - AWS EventBridge
  - AWS S3
  - cron
subtitle: 'Use AWS Lambda to save the Tidytuesday dataset to AWS S3 every Wednesday'
summary: 'An introduction to working with AWS S3 from R and a step-by-step workflow to set an AWS Lambda functon to save datasets to an S3 bucket on a weekly schedule'
authors: [teo]
lastmod: '2023-03-09T10:22:06-06:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: ['r2lambda']
show_related: true
---


## Overview

At the end of this tutorial, we would have created an AWS Lambda function that fetches
the most-recent Tidytuesday dataset and writes it into an S3 Bucket every Wednesday.
To do this, we'll first work interactively with `{r2lambda}` and `{paws}` to go 
through all the steps the Lambda function would eventually need to do, then wrap
the code and deploy it to AWS Lambda, and finally schedule it to run weekly. 

## Getting started with AWS Simple Storage Service (S3) from R

```{r}
library(r2lambda)
library(tidytuesdayR)
```

As with any AWS service supported by `{paws}`, we can easily connect to S3 and 
perform some basic operations. Below, we establish an S3 service using `r2lambda::aws_connect`,
then create a bucket called `tidytuesday-dataset`, drop and then delete and empty file, and 
delete the bucket altogether. This exercise is not very meaningful beyond learning
the basics on how to interact with S3 from `R`. Eventually, though, our lambda function
would need to do something similar, so being familiar with the process in an interactive
session helps.

**To run any of the code below, you need some environmental variables set. See 
the [Setup](https://github.com/discindo/r2lambda#build-a-docker-image-for-the-lambda-function) 
section in the `{r2lambda}` package readme for more details**

```{r}
s3_service <- aws_connect("s3")

# create a bucket on S3
s3_service$create_bucket(Bucket = "a-unique-bucket")

# upload an object to our bucket
tmpfile <- tempfile(pattern = "object_", fileext = "txt")
write("test", tmpfile)
(readLines(tmpfile))
s3_service$put_object(Body = tmpfile, Bucket = "a-unique-bucket", Key = "TestFile")

# list the contents of a bucket
s3_service$list_objects(Bucket = "a-unique-bucket")

# delete an object from a bucket
s3_service$delete_object(Bucket = "a-unique-bucket", Key = "TestFile")

# delete a bucket
s3_service$delete_bucket(Bucket = "a-unique-bucket")
```

Now, the above procedure used a local file, but what if we generated some data during
our session, and we want to stream that directly to S3 without saving to file? In
many cases, we don't have the option to write to disk or simply don't want to.

In such cases we need to serialize our data object before trying to `put` it in the bucket.
This comes down to calling `serialize` with `connection=NULL` to generate a `raw` 
vector without writing to a file. We can then put the `iris` data set from memory into our 
`a-unique-bucket` S3 bucket.

```{r}
s3_service <- aws_connect("s3")

# create a bucket on S3
s3_service$create_bucket(Bucket = "a-unique-bucket")

# upload an object to our bucket
siris <- serialize(iris, connection = NULL)
s3_service$put_object(Body = siris, Bucket = "a-unique-bucket", Key = "TestFile2")

# list the contents of a bucket
s3_service$list_objects(Bucket = "a-unique-bucket")

# delete an object from a bucket
s3_service$delete_object(Bucket = "a-unique-bucket", Key = "TestFile2")

# delete a bucket
s3_service$delete_bucket(Bucket = "a-unique-bucket")
```

OK. With that, we now know the two steps our Lambda function would need to do:

  1. fetch the most recent Tidytuesday data set (see 
  [this post](https://discindo.org/post/an-r-aws-lambda-function-to-download-tidytuesday-datasets/) for details)
  2. put the data set as an object in the S3 bucket
  
Still in an interactive session, lets just write the code that our Lambda would have
to execute.

```{r}
library(tidytuesdayR)

# Find the most recent tuesday and fetch the corresponding data set
most_recent_tuesday <- tidytuesdayR::last_tuesday(date = Sys.Date())
tt_data <- tidytuesdayR::tt_load(x = most_recent_tuesday)

# by default it comes as class `tt_data`, which causes problems
# with serialization and conversion to JSON. So best to extract
# the data set(s) as a simple list
tt_data <- lapply(names(tt_data), function(x) tt_data[[x]])

# then serialize
tt_data_raw <- serialize(tt_data, connection = NULL)

# create a bucket on S3
s3_service <- r2lambda::aws_connect("s3")
s3_service$create_bucket(Bucket = "tidytuesday-datasets")

# upload an object to our bucket
s3_service$put_object(
  Body = tt_data_raw, 
  Bucket = "tidytuesday-datasets", 
  Key = most_recent_tuesday
)

# list the contents of our bucket and find the Keys for all objects
objects <- s3_service$list_objects(Bucket = "tidytuesday-datasets")
sapply(objects$Contents, "[[", "Key")
#> [1] "2023-03-07"

# fetch a Tidytuesday dataset from S3
tt_dataset <- s3_service$get_object(
  Bucket = "tidytuesday-datasets", 
  Key = most_recent_tuesday
)

# convert from raw and show the first few rows
tt_dataset$Body %>% unserialize() %>% head()
```

Now we should have everything we need to write our Lambda function.

## Lambda + S3 integration: Dropping a file in an S3 bucket

Wrapping the above interactive code into a function and also, defining an `s3_connect`
function as a helper to create an S3 client within the function. By doing this, we
avoid adding `r2lambda` as a dependency to the Lambda function. (At the time of writing,
`r2lambda` does not yet support non-CRAN packages.)

```{r}
s3_connect <- function() {
  paws::s3(config = list(
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

tidytuesday_lambda_s3 <- function() {
  most_recent_tuesday <- tidytuesdayR::last_tuesday(date = Sys.Date())
  tt_data <- tidytuesdayR::tt_load(x = most_recent_tuesday)
  tt_data <- lapply(names(tt_data), function(x) tt_data[[x]])
  tt_data_raw <- serialize(tt_data, connection = NULL)
  
  s3_service <- s3_connect()
  s3_service$put_object(Body = tt_data_raw,
                        Bucket = "tidytuesday-datasets",
                        Key = most_recent_tuesday)
  
}
```

Now, calling `tidytuesday_lambda_s3()` should fetch and put the most recent 
Tidytuesday data set into our S3 bucket. To test it, we run:

```{r}
tidytuesday_lambda_s3()

list_objects <- function(bucket) {
  s3 <- s3_connect()
  obj <- s3$list_objects(Bucket = bucket)
  sapply(obj$Contents, "[[", "Key")
}

list_objects("tidytuesday-datasets")
#> [1] "2023-03-07"
```

On to the next step, to create and deploy the Lambda function. We have a few 
considerations here:

1. For the Lambda function to connect to S3, it needs access to some environmental 
variables. The same ones as we have in our current interactive session without which
we can't establish local clients of AWS services. These are: `REGION`, `PROFILE`,
`SECRET_ACCESS_KEY`, and `ACCESS_KEY_ID`. To include these envvars in the Lambda
docker image on deploy, use the `set_aws_envvars` argument of `deploy_lambda`.

2. We have some dependencies that would need to be available in the docker image. 
We already saw how to install `{tidytuesdayR}` in our Lambda docker image in a 
[previous post](https://discindo.org/post/an-r-aws-lambda-function-to-download-tidytuesday-datasets/).
Besides this, we also need to install `{paws}`, because without it we can't interact
with S3. To do this, we just need to add `dependencies = c("tidytuesdayR", "paws")` 
when building the image with `r2lambda::build_lambda`.

### Build

```{r}
r_code <- "
  s3_connect <- function() {
    paws::s3(config = list(
      credentials = list(
        creds = list(
          access_key_id = Sys.getenv('ACCESS_KEY_ID'),
          secret_access_key = Sys.getenv('SECRET_ACCESS_KEY')
        ),
        profile = Sys.getenv('PROFILE')
      ),
      region = Sys.getenv('REGION')
    ))
  }
  
  tidytuesday_lambda_s3 <- function() {
    most_recent_tuesday <- tidytuesdayR::last_tuesday(date = Sys.Date())
    tt_data <- tidytuesdayR::tt_load(x = most_recent_tuesday)
    tt_data <- lapply(names(tt_data), function(x) tt_data[[x]])
    tt_data_raw <- serialize(tt_data, connection = NULL)
    
    s3_service <- s3_connect()
    s3_service$put_object(Body = tt_data_raw,
                          Bucket = 'tidytuesday-datasets',
                          Key = most_recent_tuesday)
  }
  
  lambdr::start_lambda()
"

tmpfile <- tempfile(pattern = "tt_lambda_s3_", fileext = ".R")
write(x = r_code, file = tmpfile)
```

```{r}
runtime_function <- "tidytuesday_lambda_s3"
runtime_path <- tmpfile
dependencies <- c("tidytuesdayR", "paws")

r2lambda::build_lambda(
  tag = "tidytuesday_lambda_s3",
  runtime_function = runtime_function,
  runtime_path = runtime_path,
  dependencies = dependencies
)
```

### Deploy

We set a generous 2 minute timeout, just to be safe that the data set is successfully
copied to S3. And we also increase the available memory to 1024 mb. Note also the 
flag to pass along our local AWS envvars to the deployed lambda environment.

```{r}
r2lambda::deploy_lambda(
  tag = "tidytuesday_lambda_s3",
  set_aws_envvars = TRUE,
  Timeout = 120,
  MemorySize = 1024)
```

### Invoke

We invoke as usual, with an empty list as payload because our function does not 
take any arguments.

```{r}
r2lambda::invoke_lambda(
  function_name = "tidytuesday_lambda_s3", 
  invocation_type = "RequestResponse", 
  payload = list(),
  include_logs = TRUE)

#> INFO [2023-03-08 23:50:46] [invoke_lambda] Validating inputs.
#> INFO [2023-03-08 23:50:46] [invoke_lambda] Checking function state.
#> INFO [2023-03-08 23:50:47] [invoke_lambda] Function state: Active.
#> INFO [2023-03-08 23:50:47] [invoke_lambda] Invoking function.
#> 
#> Lambda response payload: 
#> {"Expiration":[],"ETag":"\"4f5a6085215b9074faed28d816696a99\"","ChecksumCRC32":[],
#> "ChecksumCRC32C":[],"ChecksumSHA1":[],"ChecksumSHA256":[],"ServerSideEncryption":"AES256",
#> "VersionId":[],"SSECustomerAlgorithm":[],"SSECustomerKeyMD5":[],"SSEKMSKeyId":[],
#> "SSEKMSEncryptionContext":[],"BucketKeyEnabled":[],"RequestCharged":[]}
#> 
#> Lambda logs: 
#> OpenBLAS WARNING - could not determine the L2 cache size on this system, assuming 256k
#> INFO [2023-03-09 05:50:49] Using handler function  tidytuesday_lambda_s3
#> START RequestId: c6cb0600-3400-4ca3-9232-8af53542f8e8 Version: $LATEST
#> --- Compiling #TidyTuesday Information for 2023-03-07 ----
#> --- There is 1 file available ---
#> --- Starting Download ---
#> Downloading file 1 of 1: `numbats.csv`
#> --- Download complete ---
#> END RequestId: c6cb0600-3400-4ca3-9232-8af53542f8e8
#> REPORT RequestId: c6cb0600-3400-4ca3-9232-8af53542f8e8	Duration: 12061.06 ms	
#> Billed Duration: 13331 ms	Memory Size: 1024 MB	Max Memory Used: 181 MB	Init 
#> Duration: 1269.59 ms	
#> SUCCESS [2023-03-08 23:51:01] [invoke_lambda] Done.
```

Then, to confirm that a Tidytuesday data set was written to S3 as an object in the 
bucket `tidytuesday-datasets` we would run:

```{r}
s3_service <- r2lambda::aws_connect(service = "s3")
objs <- s3_service$list_objects(Bucket = "tidytuesday-datasets")
objs$Contents[[1]]$Key
#> [1] "2023-03-07"
```

We expect to see one object with a `Key` matching the date of the most recent Tuesday.
At the time of writing that is March 7, 2023.

### Schedule

Finally, to copy the Tidytuesday dataset on a weekly basis, for example, every Wednesday,
we would use `r2lambda::schedule_lambda` with an execution rate set by `cron`.

First, to validate that things are working, we can set the lambda on a 5-minute
schedule and check the time stamp on the on the S3 object to make sure it is updated
every 5 minutes:

```{r}
# schedule the lambda to execute every 5 minutes
r2lambda::schedule_lambda(
  lambda_function = "tidytuesday_lambda_s3", 
  execution_rate = "rate(5 minutes)"
  )

# occasionally query the S3 bucket status and the LastModified time stamp
objs <- s3_service$list_objects(Bucket = "tidytuesday-datasets")
objs$Contents[[1]]$LastModified
```

If all is well, set it to run every Wednesday at midnight:

```{r}
r2lambda::schedule_lambda(
  lambda_function = "tidytuesday_lambda_s3",
  execution_rate = "cron(0 0 * * Wed *)"
  )
```

Next Wednesday morning, we should have two objects, with keys matching the two 
most-recent Tuesdays.

## Summary
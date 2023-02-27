---
title: Set an R-based AWS function to run on a schedule
author: 'teo'
date: '2023-02-26'
slug: set-an-r-based-aws-function-to-run-on-a-schedule
categories:
  - AWS
  - R
tags:
  - r2lambda
  - AWS Lambda
  - AWS EventBridge
  - AWS CloudWatch
  - cron
subtitle: ''
summary: 'A step-by-step tutorial on how to deploy an R-based AWS Lambda function, how to set it to run on a recurring schedule, how to validate that the execution happens on schedule, and how to clean up. All from the R console.'
authors: [teo]
lastmod: '2023-02-26T18:28:28-06:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: ['r2lambda']
show_related: true
---

A common use of the AWS Lambda service is to set the cloud functions to run on a 
recurring schedule, e.g. to collect logs, move data, or perform some ETL process.
In this post, we'll see how we can set up an AWS Lambda function, running `R`, on 
a schedule.

## A lambda runtime function

We start with a simple function that does not require any input and does not return
anything. If this example lambda is to run on a schedule, we don't want to worry
about any input arguments. Also, we want this lambda function to simply have a 
side-effect, like printing something to the logs, without returning any data or writing
to a database. This will help us greatly with the setup, in that we'll be able deploy 
and schedule the lambda with minimal involvement from other AWS services.

With this in mind, we have the following function that simply prints the system time.
Printing the current time makes sense because we can easily check that the lambda runs
on the correct schedule from the logs.

```{r}
current_time <- function() {
  print(paste("CURRENT TIME: ", Sys.time()))
}
```

## Build, test, and deploy

Here, we follow the procedure described in `Tidy Tuesday dataset Lambda` vignette.
We write this to a file that we'll use to build the lambda `docker` image:

```{r}
r_code <- "
  current_time <- function() {
    print(paste('CURRENT TIME:', Sys.time()))
  }
  
  lambdr::start_lambda()
"

tmpfile <- tempfile(pattern = "current_time_lambda_", fileext = ".R")
write(x = r_code, file = tmpfile)
```

And then build the `docker` image. Note that we don't have any dependencies other 
than base `R`.

```{r}
r2lambda::build_lambda(
  tag = "current_time",
  runtime_function = "current_time",
  runtime_path = tmpfile,
  dependencies = NULL
)
```

We test the lambda docker container locally, because it makes sense. The console 
output should include the log messages and the standard output string showing the
current time.

```{r}
r2lambda::test_lambda(tag = "current_time", payload = list())
```

Then, we deploy the lambda to AWS, leaving the lambda environment to its defaults, 
as 3 seconds should be enough to get and print the current time.

```{r}
r2lambda::deploy_lambda(tag = "current_time")
```

Finally, we invoke the cloud instance of our function, to make sure everything went 
well. Be sure to include the logs, as this particular function does not return anything.

```{r}
r2lambda::invoke_lambda(
  function_name = "current_time",
  invocation_type = "RequestResponse",
  payload = list(),
  include_logs = TRUE
)
```

## Schedule to run every minute

To make a lambda function run on a recurring schedule, we need to update an already 
deployed function. This involves three steps and two AWS services, Lambda and EventBridge:

- creating a schedule event role (EventBridge, `paws::eventbridge`)
- adding permissions to this role to invoke lambda functions (Lambda, `paws::lambda`)
- adding our target lambda function to eventBridge

Detailed instructions are available in the [AWS documentation](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-run-lambda-schedule.html).
The function `schedule_lambda` abstracts these three steps in one go. To set a lambda 
on a schedule, we need the name of the function we wish to update, and the rate at which
we want EventBridge to invoke it. Two rate-setting expression formats are supported, 
`cron` and `rate`. For example, to schedule a lambda to run every Sunday at midnight, 
we could use `execution_rate = "cron(0 0 * * Sun)"`. Alternatively, to schedule a lambda
to run every 15 minutes, we might use `execution_rate = "rate(15 minutes)"`. The details are
in this [article](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html)

```{r}
r2lambda::schedule_lambda(lambda_function = "current_time", execution_rate = "rate(1 minute)")
```

## Checking the AWS logs

To see if our function runs every minute, we can take a look at the AWS logs. If the 
function was writing to a database, or dropping files in an S3 bucket, we could also check 
the contents of those resources for the effects of the scheduled lambda function. But as
our example function only prints the current time, the only way to know that it indeed runs
every minute is to check the logs.

To do this, we'll use `paws` and `r2lambda::aws_connect` to establish an AWS CloudWatchLogs
service locally, and fetch the recent logs to look for traces of our lambda function.


In the first step, we connect to `cloudwatchlogs` and fetch the names of the log groups.
Inspect the `logs` object below to find the name corresponding to the lambda function
whose logs we want to fetch.

```{r}
logs_service <- r2lambda::aws_connect(service = "cloudwatchlogs")
logs <- logs_service$describe_log_groups()
(logGroups <- sapply(logs$logGroups, "[[", 1))
```

Then, we can grab only the data for our scheduled lambda function:

```{r}
current_time_lambda_logs <- logs_service$filter_log_events(
  logGroupName = "/aws/lambda/current_time")
```

And pull only the message printed by our `R` function wrapped in the lambda:

```{r}
messages <- sapply(current_time_lambda_logs$events, "[[", "message")
current_time_messages <- messages[grepl("CURRENT TIME", messages)]
data.frame(Current_time_lambda = current_time_messages)

#>                         Current_time_lambda
#> 1 [1] "CURRENT TIME: 2023-02-26 22:53:55"\n
#> 2 [1] "CURRENT TIME: 2023-02-26 22:54:41"\n
#> 3 [1] "CURRENT TIME: 2023-02-26 22:55:41"\n
#> 4 [1] "CURRENT TIME: 2023-02-26 22:56:41"\n
#> 5 [1] "CURRENT TIME: 2023-02-26 22:57:41"\n

```

## Clean up

We don't want to let a this trivial lambda fire every minute, it will still incur 
some cost. So its wise to delete the event schedule rule and maybe even the lambda 
function it self.

To remove the event rule, we first need to remove associated targets, and then remove
the rule.

```{r}
# connect to the EventBridge service
events_service <- r2lambda::aws_connect("eventbridge")
# find the names of all rules we need
schedule_rules <- events_service$list_rules()[[1]] %>% sapply("[[", 1)

# find the targets associated with the rule we want to remove
rule_to_remove <- schedule_rules[[1]]

target_arn_to_remove <- events_service$list_targets_by_rule(Rule = rule_to_remove)$Targets[[1]]$Id
events_service$remove_targets(Rule = rule_to_remove, Ids = target_arn_to_remove)
events_service$delete_rule(Name = rule_to_remove)

events_service$list_rules()[[1]] %>% sapply("[[", 1)

```

Finally, to remove the Lambda:

```{r}
lambda_service <- r2lambda::aws_connect("lambda")
lambda_service$list_functions()$Functions %>% sapply("[[","FunctionName")
lambda_service$delete_function(FunctionName = "parity1")
```

## Summary

In this post: 
  - we wrote a simple lambda runtime function, 
  - built a docker image locally,
  - tested the lambda invocation, 
  - deployed it to AWS Lambda, 
  - updated it to run on a schedule,
  - checked the AWS logs to confirm it executes at the correct times, and 
  - cleaned up our AWS environment. 

I hope you found this tutorial useful, and that it will motivate you to try the `r2lambda` 
package. I am looking for feedback on whether or not the workflows from `r2lambda` are
working for other people -- not many have tried it so far. It would also be very interesting
to hear suggestions on how to improve the interface, what features to add, what 
additional documentation to include, and so on. Try it and share your experience!



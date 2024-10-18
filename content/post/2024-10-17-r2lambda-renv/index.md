---
title: {r2lambda} update to support multi-file projects and {renv} projects
author: 'Teo'
date: '2024-10-17'
slug: r2lambda-renv
categories:
  - AWS
  - R
tags:
  - r2lambda
  - AWS Lambda
  - renv
subtitle: ''
summary: 'A demo of two newly added features in the `{r2lambda}` R package. How to deploy R projects with multiple files, and how to use the `{renv}` lockfile to manage dependencies in the AWS Lambda docker image'
authors: [teo]
lastmod: '2024-10-17T21:39:39-06:00'
featured: yes
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: ['r2lambda']
show_related: true
---


## Deploy a project with multiple R scripts and `{renv}`-managed environment to AWS Lambda

It has been a while since I've had the chance to work on my `{r2lambda}` 
project. In particular, there were a couple of good points made by a user on 
GitHub about functionality that is missing from the package. The option to 
deploy multiple files, e.g., one runtime function that depends on helpers in
the same project organized in different files. And another, to enable `{renv}`
management of the `R` environment within the AWS Lambda docker image. Both 
excellent points that I wished I addressed earlier. But better late than never.

Both of these features required minor adjustments to the codebase. Copying 
additional supports scripts and restoring the `{renv}` environment should both
happen when the AWS Lambda docker image is built, so the logic to create the 
Dockerfile needed to be updated. Accordinly, the `r2lambda::build_lambda` 
function now has two additional arguments:

```
#' @param support_path path to the support files (if any). Either NULL 
#' (the default) if all needed code is in the same `runtime_path` script, or a 
#' character vector of paths to additional files needed by the runtime script.
#' @param renvlock_path path to the renv.lock file (if any). Default is NULL.
#' 
#' @details Use either `renvlock_path` or `dependencies` to install required
#' packages, not both. By default, both are `NULL`, so the Docker image will
#' have no additional packages installed.
```

To include any support scripts, provide a character vector script paths to 
the `support_path` argument when building the Lambda docker image locally with
`build_lamdba`. 

[Note that, multi-file project was supported previously as well,
although perhaps not explicitly. An approach that I like is to create an 'R' 
package that exports the runtime function needed for the Lambda. Then one just
needs to make that custom `R` package a dependency of the project and either
install in the AWS Lambda docker image it through `dependencies` or 
`renvlock_path`.]

To use an existing `renv.lock` for installation of dependencies, provide its 
path to the `renvlock_path` argument to `build_lambda`. This instructs the code
to copy the `renv.lock` file to the image and run `renv::restore()` which will 
reconstruct the `R` environment inside the docker image. I really like this 
feature, as it minimizes the size of the Dockefile and removes some potential
headaches with R package dependencies from different repositories (CRAN, 
BioConductor, GitHub, etc).

## Demo code

Assuming we have a folder with the following structure:

```
~/Desktop$ ls -1 iris-lambda/
renv/
renv.lock
runtime.r
support.r
test-code.r
```

Where, `support.r` defines some function that `runtime.r` uses for the Lambda:

```
get_iris_summary_by_species <- function(species) {
    iris |>
    dplyr::filter(Species == species) |>
    dplyr::summarise(
      mean = mean(Sepal.Length),
      sd = sd(Sepal.Length)
    )
}
```

Then `runtime.r`, sources the support script, and calls the function defined 
there:

```
source("support.r")

iris_summary <- function(species) {
  get_iris_summary_by_species(species)
}

lambdr::start_lambda()
```

Then the following should work, passing the support script and renv.lock to 
r2lambda::build_lambda:

```
dir("~/Desktop/iris-lambda")
runtime_function <- "iris_summary"
runtime_path <- "~/Desktop/iris-lambda/runtime.r"
support_path <- "~/Desktop/iris-lambda/support.r"
renvlock_path <- "~/Desktop/iris-lambda/renv.lock"
dependencies <- NULL

# Might take a while, its building a docker image
build_lambda(
  tag = "my_iris_lambda",
  runtime_function = runtime_function,
  runtime_path = runtime_path,
  support_path = support_path,
  renvlock_path = renvlock_path,
  dependencies = dependencies
)

# test
payload <- list(species = "setosa")
tag <- "my_iris_lambda"
test_lambda(tag = tag, payload)


# deploy

# Might take a while, its pushing it to a remote repository
deploy_lambda(
  tag = "my_iris_lambda",
  Timeout = 30
)

invoke_lambda(
  function_name = "my_iris_lambda",
  invocation_type = "RequestResponse",
  payload = list(species = "versicolor"),
  include_logs = FALSE
)

invoke_lambda(
  function_name = "my_iris_lambda",
  invocation_type = "RequestResponse",
  payload = list(species = "setosa"),
  include_logs = FALSE
)
``` 
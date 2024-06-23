---
title: Few notes on getting R package data from the local library
author: novica
date: '2024-06-23'
slug: few-notes-on-getting-r-package-data-from-the-local-library
categories:
  - R
tags: - packages
subtitle: ''
summary: 'The missing conventions in packages' description'
authors: [novica]
lastmod: '2024-06-23T10:29:32+02:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---


I am involved in a [Posit Team](https://posit.co/products/enterprise/team/) 
deployment, and one of the things that we are looking into is default R packages
that should be made available to all users. We are looking to do this because we
would like to avoid people installing, for example `tidyverse`, in their own local
libraries in order to save on space and to make sure everyone is on the same 
version, at least for the packages that are considered to be a preferred 
default option for working with data in R.

In order to do this we wanted to collect all of the packages that are currently
used, their versions, source repository and similar information.  That way we 
can see if anything else should be installed for all users,  in addition to the 
best guess that we should have `tidyverse`, `tidymodels`, and `shiny`.

In order to do this we first have to get the list of installed packages, which
is fairly simple to do:

```
installed_packages <- installed.packages()
```

Then, `utils::packageDescription` can be used to get the packages` descriptions.
For example for getting the package description for `dplyr` we can run:

```
dplyr_pkg_desc <- utils::packageDescription('dplyr')
```

The result is a list, and it can be subsetted to see details, for example:

```
> dplyr_pkg_desc[1]
$Type
[1] "Package"

> dplyr_pkg_desc[2]
$Package
[1] "dplyr"

> dplyr_pkg_desc[3]
$Title
[1] "A Grammar of Data Manipulation"

> dplyr_pkg_desc[4]
$Version
[1] "1.1.4"
```

At this point, I am thinking that all these description files have the same
structure. Therefore if I want to get all packages' version i need to `lapply` to
get the fourth element and that's that. It turns out this is not entirely true.
Not all packages have the same structure of the description. See `tydir`:

```
> tidyr_pkg_desc[1]
$Package
[1] "tidyr"

> tidyr_pkg_desc[2]
$Title
[1] "Tidy Messy Data"

> tidyr_pkg_desc[3]
$Version
[1] "1.3.1"

> tidyr_pkg_desc[4]
$`Authors@R`
[1] "c(\n    person(\"Hadley\", \"Wickham\", , \"hadley@posit.co\", role = c(\"aut\", \"cre\")),\n    person(\"Davis\", \"Vaughan\", , \"davis@posit.co\", role = \"aut\"),\n    person(\"Maximilian\", \"Girlich\", role = \"aut\"),\n    person(\"Kevin\", \"Ushey\", , \"kevin@posit.co\", role = \"ctb\"),\n    person(\"Posit Software, PBC\", role = c(\"cph\", \"fnd\"))\n  )"
```

Number four is the authors, and version is three. And these are two packages that
are ultimately from the same author. Look at `data.table`:

```
> data.table_pkg_desc[1]
$Package
[1] "data.table"

> data.table_pkg_desc[2]
$Version
[1] "1.15.4"

> data.table_pkg_desc[3]
$Title
[1] "Extension of `data.frame`"

> data.table_pkg_desc[4]
$Depends
[1] "R (>= 3.1.0)"
```

Now, of course, subsetting works with using the name, instead of the position:

```
> dplyr_pkg_desc[["Version"]]
[1] "1.1.4"
> tidyr_pkg_desc[["Version"]]
[1] "1.3.1"
> data.table_pkg_desc[["Version"]]
[1] "1.15.4"
```

However, to be honest, it rarely comes to my mind to subset lists like this. 

```
package_names <- installed.packages()[, 1]

all_packages_data <- lapply(package_names,  utils::packageDescription)

version_number <-
  lapply(1:length(package_names), function (x) {
    all_packages_data[[x]][["Version"]]
  })
```

The above is possible, and then to `cbind` all needed fields in a `data.frame`.

However, looking at the `packageDescription` documentation, it seems the best way 
is to use additional arguments the function. This is neat:

```
package_data <- lapply(
  package_names,
  utils::packageDescription,
  fields = c("Package", "Version", "Built", "Repository")
) 
```

And then there is another surprise. The results are with class
`packageDescription` which makes getting to a `data.frame`, or `tibble` in 
this case, a bit complicated:

```
package_data <- purrr::map_df(
  package_names,
  utils::packageDescription,
  fields = c("Package", "Version", "Built", "Repository")
) 
Error in `as_tibble()`:
! All columns in a tibble must be vectors.
✖ Column `askpass` is a `packageDescription` object.
```

The full solution involves a step of changing the class of the object using
`as`, and then reassigning the names of each element, because the previous step 
removes them:

```
package_data <- lapply(
  package_names,
  utils::packageDescription,
  fields = c("Package", "Version", "Built", "Repository")
) |> 
  lapply(as, Class = "list") |> 
  lapply(setNames, c("Package", "Version", "Built", "Repository")) |> 
  dplyr::bind_rows()
```

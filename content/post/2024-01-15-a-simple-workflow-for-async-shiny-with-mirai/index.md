---
title: A simple workflow for async {shiny} with {mirai}
author: "teo"
date: "2024-01-15"
slug: a-simple-workflow-for-async-shiny-with-mirai
categories: [R, shiny, async programming, mirai]
tags: [daemons, parallelization, shiny module]
subtitle: ""
summary: "An module-based approach to simplify async calls in `{shiny}` apps using `{mirai}`"
authors: [teo]
lastmod: "2024-01-15T13:10:09-06:00"
featured: no
image:
  caption: ""
  focal_point: ""
  preview_only: no
projects: []
---

In my [previous post](https://discindo.org/post/an-opiniated-workflow-for-async-shiny-with-callr/),
I developed a `shiny` module to encapsulate the logic of sending and monitoring background async
tasks. The main advantage of this approach was to simplify making repeated async calls
in larger applications. In the first version of this module, the async process was
created with `callr:r_bg`, an approach that [my self](https://discindo.org/post/asynchronous-execution-in-shiny/) and
[others](https://hypebright.nl/index.php/2023/09/12/async-programming-in-shiny-with-crew-and-callr/) have used before.

However, there is one, potentially significant, drawback of using `callr` in such
a way. Take this hypotetical scenario as an example. You have a shiny app with
five async tasks triggered in response to a user changing a dataset. You test it locally,
and everything works great. Then you deploy and share with the world. Ten of your
followers click on the link more-or-less at the same time and visit the application,
each choosing one of three datasets available in your data science app. The app's
`server`, featuring `async` functions gets to work, and initializes 5 (tasks) \* 10 (users)
= 50 `callr::r_bg` calls, each running in a separate child R process. Some of these
copy nothing the child enviroment, some only a few small objects, but others a large
data object needed for the async function to transform or run a model. It should be no surprise
if the app is no longer that fast. The hosting server, even with a fast, multi-thread
processor, still hast to contend with many `R` processes and the `shiny` session
is also getting a bit bogged down, as it has potentially dozens of observers monitoring
background processes. Clearly, we need to rethink our approach.

Wouldn't it be great if we had a way to limit the total number of concurrent
child `R` processes that our `shiny` session would spawn, and have a queue system
that would start another background job as soon as one completes? Enter
[`mirai`](https://github.com/shikokuchuo/mirai). `mirai` lets us initialize a set
number of `R` `daemons` (persistent background processes) that are
ready to receive `mirai` requests and ensures FIFO (first in, first out) scheduling.
Using `mirai`, we can handle a large number of async background jobs elegantly
without overburdening the system. If the number of jobs requested by the `shiny`
app exceeds the number of available `daemons`, `mirai` would hold the jobs until
one of the daemons (threads) frees up and submit on a first-come, first-serve
basis. Just great!

## So how does it work?

For example setups for `shiny`, check out the documentation, where you can read
about [`mirai`-only](https://shikokuchuo.net/mirai/articles/shiny.html#shiny-example-usage)
solutions, as well as approaches combining
[`mirai` with `promises`](https://shikokuchuo.net/mirai/articles/shiny.html#example-using-promises).

For my application, I'll adapt the `callr` approach I described in my
[previous post](https://discindo.org/post/an-opiniated-workflow-for-async-shiny-with-callr/)
to work with `mirai`. In fact, there very little to change to make the `callr`
example work with `mirai`:

1. Change the `async` version of our function to use `mirai`

```{r}
head_six <- function(x, sleep) {
  Sys.sleep(sleep)
  head(x)
}

head_six_async_mirai <- function(x, sleep) {
  args <- list(head_six = head_six, x = x, sleep = sleep)
  bg_process <- mirai::mirai(.expr = head_six(x, sleep), .args = args)
  return(bg_process)
}
```

2. Change the polling logic in the module's server to use `mirai::unresolved`,
   rather than the `is_alive` method of the `callr` process object.

```{r}
mod_async_srv_mirai <- function(id, fun_async, fun_args, wait_for_event = FALSE, verbose = FALSE) {
  moduleServer( id, function(input, output, session){
    res_rct <- shiny::reactiveVal(NULL)
    poll_rct <- shiny::reactiveVal(TRUE)

    if (isTRUE(wait_for_event)) {
      poll_rct(FALSE)
    }

    bg_job <- reactive({
      req(isTRUE(poll_rct()))
      do.call(fun_async, fun_args)
    }) |> bindEvent(poll_rct())

    observe({
      req(isTRUE(poll_rct()))
      invalidateLater(250)
      if (verbose) {
        message(sprintf("checking: %s", id))
      }

      alive <- mirai::unresolved(bg_job())
      if (isFALSE(alive)) {
        res_rct(bg_job()$data)
        if (verbose) {
          message(sprintf("done: %s", id))
        }
        poll_rct(FALSE)
      }
    })

    return(list(
      start_job = function() poll_rct(TRUE),
      get_result = reactive(res_rct())
    ))

  })
}
```

3. In the app's `server`, or better yet `global.R` or equivalents, we need to
   initialize the `daemons`:

```{r}
  mirai::daemons(2L)
  onStop(function() mirai::daemons(0L))
```

In this setup, our shiny can run up to two parallel async jobs handled by the
`mirai` queue. These `daemons` are shared across _all users_ of our application,
irrespective of the `shiny` session. This is because `mirai`'s daemons apply to
the entire `R` session, not individual `shiny` sessions.

## Gist

For a running example of `mirai` async with the module, visit this gist:

<script src="https://gist.github.com/teofiln/59e69133e2ce08597b071ffa1cad5dc9.js"></script>

## Summary

In this post I went over an approach to organize `mirai` background async jobs using
a `shiny` module, in order to make the async code faster to write, less error prone
and overall cleaner.

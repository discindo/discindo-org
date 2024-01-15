---
title: A simple workflow for async {shiny} with {callr}
author: "teo"
date: "2024-01-12"
slug: an-opiniated-workflow-for-async-shiny-with-callr
categories: [R, shiny, async programming, callr]
tags: [shiny module]
subtitle: ""
summary: "An approach to simplify and standardize async calls in `{shiny}` apps using `{callr}`"
authors: [teo]
lastmod: "2024-01-12T00:06:30-06:00"
featured: yes
image:
  caption: ""
  focal_point: ""
  preview_only: no
projects: []
---

In the `R/Shiny` community we are fortunate to have several approaches for async programming.
It is an active field of development with a variety of
options depending on the needs of the application. For examples and deeper overviews of
the state of async programming in `R`, head over to Veerle van Leemput's
[writing](https://hypebright.nl/index.php/2023/09/05/mastering-async-programming-shiny/),
the [Futureverse documentation](https://www.futureverse.org/) or the
[mirai](https://github.com/shikokuchuo/mirai/) / [crew](https://github.com/wlandau/crew) repos.

In this post, I am going to focus on an approach to simplify making multiple
async calls in `shiny` applications. Really, it boils down to developing a module
that wraps the initialization and polling of a `callr::r_bg` process into a single
function, and makes it easier write a larger async-capable `shiny` app while
keeping the code a bit shorter, and more compact.

## The problem

I am working on refactoring a relatively large `shiny` application where many of the
computations are time-consuming. Ideally, I would like to convert the major bottlenecks
into async routines. Typically, is is done by setting up `future/promise` constructs or
sending a job to a subprocess, keeping the main `shiny` process free, and then
polling the subprocess 'manually' to fetch the result (`callr`/`mirai`/`crew`).

After reviewing the available options, and trying a few things, I decided
to go with `callr` for async, although the `mirai`, and `crew` where close seconds.
This choice was mostly because of `callr`'s simplicity and because I have [previous
experience](https://discindo.org/post/asynchronous-execution-in-shiny/) with it.

The `callr` workflow can be sumarised in the following steps:

- send a call to the subprocess (possibly within a reactive and dependent on events within `shiny`)
- monitor the status of the background process to know when to fetch the results
- the polling observer has to have a switch, so we don't waste resources on polling
  while there is nothing running.

In all, its probably some 15-20 lines of code, depending on the complexity of
the function call we are sending to the subprocess. It looks something like this:

```{r}
# The function we want to run async
# (sleep is added to mimic long computation)
head_six <- function(x, sleep) {
  Sys.sleep(sleep)
  head(x)
}

# the r_bg call
args <- list(head_six = head_six, x = my_data, sleep = 5)
bg_process <- callr::r_bg(
  func = function(head_six, x, sleep) {
    head_six(x, sleep)
  },
  args = args,
  supervise = TRUE
)

# turn on polling after the task has been sent to the subprocess
poll_switch <- shiny::reactiveVal(TRUE)

# reactive to store the result returned by the subprocess
result_rct <- shiny::reactiveVal(NULL)

# monitor the background process
shiny::observe({
  shiny::req(isTRUE(poll_switch()))
  shiny::invalidateLater(300)
  message("checking")

  alive <- bg_job()$is_alive()
  if (isFALSE(alive)) {
    res_rct(bg_job()$get_result())
    message("done")
    poll_rct(FALSE)
  }
})

# do stuff with `result_rct()`

```

Having to write this in 20 different places where async might be needed in
an application is definitelly a chore, not to mention error-prone as one needs
to keep track of the names of the process objects, polling switches, and result
reactives. Then of course, some async bits would need to respond to events, like
button clicks or other reactives in the `shiny` session, while others would need
to run without explicit triggers, adding to the complexity and maintanence of the
codebase.

## The solution

I wanted to simplify the above process and make it quicker to write the async
code. I wanted a function or a module server that would take a function by name
and its arguments and then run the function in a background process, poll the
process and return the result when ready. Additionally, I wanted this module
to be flexible enough such that one can trigger the execution from the outside
(e.g., from the parent module) or to run without external triggers.

In the end, I came up with a solution with 3 components: the function that does the
long computation, an _async_ version of this function, and a module server that
will do the `shiny` things. Bellow are the 3 parts starting with the trivial `head_six`
function (same as above):

```{r}
# The function we want to run async
# (sleep is added to mimic long computation)
head_six <- function(x, sleep) {
  Sys.sleep(sleep)
  head(x)
}
```

The async version of the function is a wrapper that is prepared manually for the
function we need to run async. It is abstracting the `callr::r_bg` call, and
can live in a separate script (together with the function it wraps) instead
of the `shiny` server. There probably are ways to generate this function with
code, and I might try that soon, but for now creating this wrapper does not
bother me much. Having an async function that you can test and debug interactivelly
might actually be preferred.

```{r}
# Async version of `head_six`
# calls `r_bg` and returns the process object
head_six_async <- function(x, sleep) {
  args <- list(head_six = head_six, x = x, sleep = sleep)
  bg_process <- callr::r_bg(
    func = function(head_six, x, sleep) {
      head_six(x, sleep)
    },
    args = args,
    supervise = TRUE
  )
  return(bg_process)
}
```

The third part is the function (module server) that calls the async version of
the function doing the time-consumig task. The module also has reactives
to switch polling on/off, and an observer to monitor and fetch the result. It
returns a list with two elements, a reactive with the result of the async
job, and a function that updates the polling reactive (`poll_rct`) that allows
one to initiate the task from the outside. For example if we had a button in
another module that should trigger the computation _inside_ this async module.

```{r}
mod_async_srv <- function(id, fun_async, fun_args, wait_for_event = FALSE) {
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
      message(sprintf("checking: %s", id))

      alive <- bg_job()$is_alive()
      if (isFALSE(alive)) {
        res_rct(bg_job()$get_result())
        message(sprintf("done: %s", id))
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

Note that this _is not_ a typical `shiny` module, in that it does not have
(and does not strictly need) a UI part. So we don't have to worry
about the namespace (`ns <- session$ns`) inside it. We simply want to observe
and return. One could add a UI component to, perhaps, notify the user about the
progress (checking, checking, ... done) of the async job.

With this module, refactoring to async becomes more streamlined. For example,
we could have a scenario like this.

```{r}
server <- function(input, output, session) {

  # async job triggered on event (input$go_async_job1)
  async_job1 <- mod_async_srv(
    id = "job1_srv",
    fun_async = "job1_async",
    fun_args = list(x = x, z = z),
    wait_for_event = TRUE
  )

  observeEvent(input$go_async_job1, {
    async_job1$start_job()
  })

  output$x <- renderPlot({
    plot_fun(async_job1$get_result())
  })

  # async job that runs without external intervention
  async_job2 <- mod_async_srv(
    id = "job2_srv",
    fun_async = "job2_async",
    fun_args = list(a = a, b = b),
    wait_for_event = FALSE
  )

  output$y <- renderPlot({
    table_fun(async_job2$get_result())
  })
}
```

Note that the two instances of `mod_async_srv` use different async functions
with different sets of arguments, and are triggered in different ways. Providing
some flexibility, while keeping the server code minimal.

Nothing special here, no magic, just some wrappers to make life a bit easier when
writing large `shiny` applications with async capabilities.

## Demo

To test out this approach you can download the following `gist`. In it, I have
two `callr` background async jobs, to show the `head` of `iris` and `mtcars`,
with different sleep time. The `iris` job waits for user click, while the
`mtcars` job runs on its own when the app starts. Neither async job blocks
the main `shiny` process, as they are both in the background, so the slider and
histogram work throughout.

<script src="https://gist.github.com/teofiln/7815c3c5197bb231b2188070593029ec.js"></script>

## Summary

In this post I went over an approach to organize `callr` background async jobs using
a module, in order to make the async code faster to write, less error prone and overall cleaner.

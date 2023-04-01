---
title: How to use Bootstrap 5 popovers in Shiny applications
author: 'teo'
date: '2023-04-01'
slug: bs5-popovers
categories: [R, Shiny]
tags: [bslib, Bootstrap 5, popover, tooltip, popper.js]
subtitle: ''
summary: 'A few tips on using bootstrap 5 popovers in Shiny'
authors: [teo]
lastmod: '2023-04-01T06:22:15-06:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

Creating custom user interfaces with `{shiny}` and `{bslib}` has never 
been easier. Using `{bslib}` it is also incredibly simple to choose and 
switch between versions of the underlying [Bootstrap](https://getbootstrap.com)
library that powers the UI, to use many of the [Bootswatch](https://bootswatch.com/) 
themes, and to create [custom components](https://rstudio.github.io/bslib/articles/custom-components.html).

One useful and commonly requested application feature are tooltips or popovers. 
These offer more detailed information or documentation to the user without
cluttering the UI with text. Historically, there have been some great 
packages that provide this functionality, for example [`{shinyBS}`](https://github.com/ebailey78/shinyBS),
and [`{bsplus}`](https://github.com/ijlyttle/bsplus). However, there are some known 
incompatibilities with newer versions of Bootstrap, and in many cases we
don't necessarily want to add these dependencies to our projects.

# Bootstrap 5 in Shiny

Using Bootstrap 5 in `{shiny}` with `{bslib}` is as easy as:

```r
bslib::page_fluid(
    title = "Test",
    theme = bslib::bs_theme(version = 5)
)
```

And adding a simple white text on dark background tooltip to an element of our page is as easy as adding a `title` attribute:

```r
ui <- bslib::page_fluid(
    title = "Test",
    theme = bslib::bs_theme(version = 5),
    htmltools::div("Welcome", title = "This is a welcome message")
)

server <- function(input, output) {}

shiny::shinyApp(ui = ui, server = server)
```

However, if we wanted to use the nice Bootstrap popovers, that
can be shown by clicking or hovering on an icon, we'd be a little disappointed. 
These don't come by default with Bootstrap 5, and they require a
bit of prep work before we can add them to our application.

# Bootstrap 5 popovers in Shiny

The requirements to enable popovers in Bootstrap 5 are well documented. Bootstrap 5 uses the [Popper](https://popper.js.org/) JavaScript library, so we have to opt-in to use them, i.e., add the Popper JS library as a dependency. To enable them everywhere in an application, we can simply include the code provided in the bootstrap documentation in our application. 

To do this in `{shiny}`, we need to define a JS callback function that will work when the HTML document is ready, along the lines of:

```js
// popovers.js
$( document ).ready(function() {

    var popoverTriggerList = [].slice.call(document.querySelectorAll('[data-bs-toggle="popover"]'))
    var popoverList = popoverTriggerList.map(function (popoverTriggerEl) {
    return new bootstrap.Popover(popoverTriggerEl)
    })
    
  });
```

Save this function to a file, say `popovers.js` and include it in our 
UI via `htmltools::includeScript` (or some other way described in the [`{shiny}` docs](https://shiny.rstudio.com/articles/packaging-javascript.html)) 
To test our setup right away, we'll also add a button that when clicked would open a popover.

```r

ui <- bslib::page_fluid(
    htmltools::includeScript("popovers.js"),
    title = "Test",
    theme = bslib::bs_theme(version = 5),
    htmltools::div("Welcome", title = "This is a welcome message"),
    htmltools::tags$button(
        type = "button",
        `data-bs-toggle` = "popover",
        title = "Popover title",
        `data-bs-content` = "Popover body", "Click me"
    )
)

server <- function(input, output) {}

shiny::shinyApp(ui = ui, server = server)

```

And we are done with the basic setup. We have a functional Bootstrap 5 popover in `{shiny}`
without adding any `R` dependencies. Next, we'll make a few minor improvements 
for ease of use and functionality. 

The default behavior of the popovers is that they are dismissed the next time we click the button (or icon)
that triggered them. This is not that great, because sometimes the icons can be small, or even hidden by 
the popover it self, so it might be hard to click and dismiss the popover. To aleviate this, we can
set the popovers to show up on hover by adding a `data-bs-trigger` = "hover" attibute.

Finally, there is some CSS conflict between `{shiny}` and the styling of the Bootstrap 5 popovers, which
causes some unecessary padding on top of the popover title. We can remove this by forcing the top-margin
on the `h3` tag to zero. Similar to before, we can add this bit of CSS to a file and include this file
as a resource in the `{shiny}` app using `htmltools::includeCSS`:

```css
/* Popover title conflict */
h3, .h3 {
  margin-top: 0  !important;
}
```

With these improvements, our basic example shiny app becomes:

```r

ui <- bslib::page_fluid(
    htmltools::includeScript("popovers.js"),
    htmltools::includeCSS("popovers.css"),
    title = "Test",
    theme = bslib::bs_theme(version = 5),
    htmltools::div("Welcome", title = "This is a welcome message"),
    htmltools::tags$button(
        type = "button",
        `data-bs-toggle` = "popover",
        `data-bs-trigger` = "hover",
        title = "Popover title",
        `data-bs-content` = "Popover body", "Click me"
    )
)

server <- function(input, output) {}

shiny::shinyApp(ui = ui, server = server)

```

# Popovers in Shiny inputs labels and tab names

The most common places where popovers are useful are next to inputs and tab names. These 
help with user experience by providing guidance and information. To create inputs and tab 
names with popovers, we'll write a function that creates icons with the popover functionality
we discussed above. 

```r
titleWithPopover <- function(title, popover_title, popover_body) {
    htmltools::span(
        class = "d-flex justify-content-between align-items-center",
        title,
        shiny::icon(
            name = "circle-info",
            style = "cursor: pointer;",
            `data-bs-toggle` = "popover",
            `data-bs-trigger` = "hover",
            title = popover_title,
            `data-bs-content` = popover_body
        )
    )
}
```

In the above function, we create a `span` with some text aligned to the left, and an icon aligned to the right,
clicking on the icon will trigger the popover. Then, to use this tag as a `label` of a shiny input, we set the
input's `label` argument to `NULL` and provide our customized label (select input example). Alternatively, we can omit the separate label altogether, and add the icon with popover to the right of an input, as would make sense for inputs that have a placeholder value (text input example).

```r
htmltools::div(
    class = "row",
    htmltools::div(
        class = "col-3",
        titleWithPopover(
            title = "Select a value",
            popover_title = "Popover title",
            popover_body = "Popover body"
        ),
        shiny::selectInput(
            inputId = "someValue",
            choices = 1:3,
            label = NULL,
            width = "100%"
        ),
        titleWithPopover(
            title = shiny::textInput(
                inputId = "textInput",
                label = NULL,
                width = "60%",
                placeholder = "Enter some text"
            ),
            popover_title = "Popover title",
            popover_body = "Popover body"
        )
    )
)
```
![Shiny inputs with icons for popover](/post/2023-04-01-bs5-popovers/screengrab1.png)

Our `titleWithPopover` function can easily be applied in other contexts too.
For example, we can create a tabset panel with tabs whose names are embelished with
icons and popovers. 

```r
bslib::navs_pill_list(
    well = TRUE,
    bslib::nav(title = titleWithPopover("Tab One",
        popover_title = "Popover title",
        popover_body = "Popover body"
    ), "Text One"),
    bslib::nav(title = titleWithPopover("Tab Two",
        popover_title = "Popover title",
        popover_body = "Popover body"
    ), "Text Two"),
    bslib::nav(title = titleWithPopover("Tab Three",
        popover_title = "Popover title",
        popover_body = "Popover body"
    ), "Text Three")
)
```

![Nav item titles with icons for popover](/post/2023-04-01-bs5-popovers/screengrab2.png)

# Summary

In this post, we went over the simple procedure to enable Bootstrap 5 popovers in a `{shiny}` application by using bits of JS and CSS for `popper.js`. We also discussed some example usage of the popovers in input labels and tab names.

# Gist

The code for a functional shiny app with popover examples is available at the following gist.

<script src="https://gist.github.com/teofiln/38b09ad2c01bea576dabea659f857bcb.js"></script>
---
title: 'Using Wikidata to draw networks of Politically Exposed Persons #2'
author: novica
date: '2022-11-17'
slug: using-wikidata-to-draw-networks-of-politically-exposed-persons-2
categories:
  - Python
tags:
  - wikidata
  - pep
  - data
  - shiny
subtitle: ''
summary: ''
authors: [novica]
lastmod: '2022-11-17T21:41:55+01:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---


In the [previous post](/post/using-wikidata-to-draw-networks-of-politically-exposed-persons-1) I outlined the approach to query data from Wikidata, 
and wrangle your way to a result about some relationships between politically exposed 
persons on the example of the rather poor data available for North Macedonia. 

Since the code is ready, and reusable, I decided wrap it into a `shiny` app for `python`. 
[Shiny](https://shiny.rstudio.com/py/) for python is still in `alpha`, so a lot of things 
may change in the future. Still it was pretty fun to write an app, and quite easy as well.

At the beginning I followed the [get started](https://shiny.rstudio.com/py/docs/get-started.html) 
tutorial and simply run:

```
pip install shiny
shiny create my_app
shiny run --reload my_app/app.py
```

This got me the basic up and I could check that things are working. 

Next, I went to the `Shiny` for `python` [examples](https://shinylive.io/py/examples/#multiple-source-files) 
page and found the example for multiple source files. 

I thought that should be the way to go because I have some functions that would probably be 
better to live outside of the main `app.py` file.

So I created a file called `functions.py` and just pasted there the code that queries the data and 
wrangles the results. All the relevant `imports` for these functions are also in this file. 

Back in the `app.py` file I added a line to import the functions:

```
from shiny import App, render, ui, reactive, req
from functions import endpoint_url, get_results, wrangle_results
```

As with Shiny for R, Shiny for Python also has the basic elements of the app in the generated file, 
so it was not difficult to change and add elements to get to an app where an input is added and 
after some work in the background a result is displayed on the page. 

In the app you can try to query North Macedonia with code `wd:Q221` or Serbia with code `wd:Q403`. However, 
Germany will crash the app since apparently [Gertrude of Süpplingenburg](https://en.wikipedia.org/wiki/Gertrude_of_S%C3%BCpplingenburg) 
born in year 1115 is a German political figure whose data of birth cannot be converted to datetime in `pandas`. :)
Probably a solution here would be to adjust the query to return only politicians who are still alive. 

The app, if it doesn't crash, returns the query and a table of the politicians sorted alphabetically 
with the relationship type and relationship name. 

Pretty neat. :)

The code for the app is available on [github](https://github.com/novica/pyshinywikidata) for anyone looking to use or improve on this. A demo shiny module is also included, just for fun. 
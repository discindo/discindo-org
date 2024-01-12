---
title: 100 days of Python and R
author: 'novica'
date: '2023-12-29'
slug: 100-days-of-python-and-r
categories:
  - R
  - python
tags:
  - R
  - python
subtitle: ''
summary: 'Random things from 100 days of code challenge.'
authors: [novica]
lastmod: '2023-12-29T18:05:13+01:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

Today, together with a friend who is looking to get into data analytics, I started 
doing the [100 days of Python](https://replit.com/learn/100-days-of-python) 
challenge at Replit.

I thought it would be a good idea to do the challenges in `R`, because why not. :)

So, I am at day 2 (whoohooo I did two challenges in one day), when I notice that
`Ctrl+Enter` for running the code in `Python` and in `R` is not the same thing. 

Challenge 2 is about user input, so in `Python` you have something like:

```
name = input("Your name: ")
email = input("Your email: ")
```

When running this with `Ctrl+Enter`, `Python` prompts for name, and waits. Once name is 
entered, prompts for email. 

In R the equivalent would be:

```
name <- readline("Your name: ")
email <- readline("Your email: ")
```

When running this with `Ctrl+Enter` in Rstudio, `R` prompts for name, but then 
enters the next line as the input for the prompt.

```
> name <- readline("Your name: ")
Your name: email <- readline("Your email: ")
```

I thought this was strange. I first thought that maybe the Replit environment is 
configured to work in such a way. But then I got the same behavior running the code
in a Jupyter notebook on my computer. 

The issue is then on R's side, or at least Rstudio's side. When running code with 
`Ctrl+Enter`, the code is being entered line by line in the terminal, causing 
the next line to be entered as input to the first `readline()`. 

However, clicking on the `Source` button to run the script, produces the behavior 
experienced with `Python`: `R` waits for the user to respond to the prompt, instead 
of entering the next line as input to the first `readline()`.

When running this with `Ctrl+Enter` in VSCode with `radian` as the 
`R` console, `R` again waits for the user to respond to the prompt.

On to challenge 3. :)
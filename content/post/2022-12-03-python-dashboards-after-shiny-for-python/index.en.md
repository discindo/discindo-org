---
title: Python dashboards after Shiny for Python
author: novica
date: '2022-12-03'
slug: python-dashboards-after-shiny-for-python
categories:
  - Python
tags:
  - dash
  - streamlit
  - app
subtitle: ''
summary: ''
authors: [novica]
lastmod: '2022-12-03T14:19:48+01:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---


After building a [demo](/post/packaging-a-python-shiny-app) in Shiny for Python I decided to see how to build a demo dashboard with the other tools available for Python, namely [Streamlit](https://streamlit.io/) and [Dash](https://dash.plotly.com/).

The good thing about doing all these in Python is that so much of the code can be reused. Both repositories are packaged as a python package, but it seems there are some limitations in how that can be used, especially in the Streamlit app. However, I am not too familiar with the python way of doing things, so I may be missing some obvious stuff. 

First, I built the Streamlit app. This was really easy. Everything about Streamlit is really friendly and I had no problems finding what I needed in the documentation. I was surprised that I was able to recreate the app in about 20 lines of code, half of what the shiny app is (not counting the functions that reside in a separate file).

The experience with Dash was a bit more involved. The structure of the code resembles shiny in the sense there are separate blocks for UI and Server (or callback as it's called in Dash). In terms of lines of code Shiny and Dash are the same -- at least for this demo. For the Dash app, I spent too much time to find out how to render a simple table. I generally found the experience of going through Dash docs a little bit confusing when compared both to Shiny and Streamlit.

I don't have any takeaways except that it was fun doing this. The code for both demos is on github ([Streamlit](https://github.com/novica/streamlit), [Dash](https://github.com/novica/dashwikidata)).
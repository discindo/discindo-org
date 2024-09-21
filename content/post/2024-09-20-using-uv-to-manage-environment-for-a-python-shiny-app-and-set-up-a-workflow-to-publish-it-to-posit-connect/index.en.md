---
title: Using uv to manage environment for a Python Shiny app and set up a GitHub action 
  to publish it to Posit Connect
author: novica
date: '2024-09-21'
slug: using-uv-to-manage-environment-for-a-python-shiny-app-and-set-up-a-workflow-to-publish-it-to-posit-connect
categories:
  - python
tags:
  - shiny
  - uv
  - github actions
subtitle: ''
summary: 'A step-by-step guide to work with a Python Shiny app with the help of 
uv and a workflow to have it published from GitHub to Posit Connect.'
authors: [novica]
lastmod: '2024-09-21T20:23:53+02:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

## It all starts with a notification

A couple of weeks ago I got a notification from LinkedIn. Unlike the usual notifications, 
this was not from an anonymous recruiter viewing my profile. It was a post by [Russ Hyde](https://www.jumpingrivers.com/authors/russ-hyde/) who was looking for examples on
how to organize the code in a Python Shiny app, and he bumped into the relatively old
[repository](https://github.com/novica/pyshinywikidata) that packages a Python Shiny app.

It was a great reminder that I should look into my previous work, as I was anyhow trying
to figure out way to deploy Python Shiny apps to Posit Connect. 

At the same time the relatively new Python packager and project manager called 
[uv](https://docs.astral.sh/uv/) caught my attention with many internet resources
about its features popping up at, more or less, at the same time (for example: 
[demo project by Damjan](https://github.com/gdamjan/uv-getting-started), 
[Unified Python packaging with uv](https://talkpython.fm/episodes/show/476/unified-python-packaging-with-uv)). 
So, I thought why not figure out how Shiny for Python 
work with `uv` and then how to make it all happen on Posit Connect.

## Step 1 in which I move the Shiny code to a new `uv` project

Setting up a new `uv` project is pretty straight forward, and their [projects guide](https://docs.astral.sh/uv/guides/projects/) make is even simpler. 

I just ran:

`$ uv init py-shiny-uv`

and I get a new folder with the following contents:

```
.
├── hello.py
├── pyproject.toml
└── README.md
```

No surprises here.

Then, since I already had all the code organized in the repo above, I just copied 
everything (including the `.git` folder and `.gitignore` file) to the `uv` project folder, 
and remove the `hello.py`. Who knows if this is a good practice, but anyway it seems
things didn't break.

## Step 2 in which I get to see some of the cool stuff `uv` can do

Next I needed a specific version of Python (my Posit Connect instance runs on 3.11.5).

First I updated my `pyproject.toml` to have the required Python version.

Then, adding a specific Python version to the project is also simple. After navigating to
the project folder, I just ran:

```
$ uv venv --python 3.11.5
```

And it gets installed in seconds. I can activate the environment in the usual way:

```
$ source .venv/bin/activate
```

And check the version:

```
(py-shiny-uv) $ python --version

3.11.5
```

This is the environment in which further development of the app could happen.

At this point I checked the git status and I have one new untracked file `.python-version`, 
which, as expected, has the Python version written inside.

# Step 3 in which I install Shiny and other dependencies for the app

As far as I recalled the app depends on three packages: `shiny`, `SPARQLWrapper`, 
and `pandas`, and I added those. 

```
(py-shiny-uv) $ uv add pandas shiny SPARQLWrapper
```

Then I just verified that those load by running python and importing them.


## Step 4 in which I create the `requirements.txt` and `manifest.json` needed for deployng on Posit Connect and things start to break

As you may know if you are doing Shiny apps, the `rsconnect-python` package is needed to 
generate the `manifest.json` file which Posit Connect uses when publishing from a git repository
(which is something that I want to do). 

Usually the way to do it is to run `rsconnect` in the folder where the app is:

```
$ rsconnect write-manifest shiny .
```

But since the idea is to use `uv` I had to try, and fail multiple times, with it.

First the problem with `rsconnect` is that it generates the files in the app directory, instead of the top level python project. Moving the files is a possibility, of course, but it seems it is an unnecessary complication.

The default way to get the requirements with `uv` is:

```
uv export -o requirements.txt
```

This however generates the dependencies with hashes, which then is a problem with the package environment not having a has. To quote the error log:

```
The editable requirement pyshinywikidata cannot be installed when requiring hashes, because there is no single file to hash.
```

Omitting the package with:

```
uv export --no-emit-project -o requirements.txt
```

Fails because now the package containing the app is no longer in the `requirements.txt`, and Posit Connect can't find the module to run.

Finally, after few more trial and errors I found the solution in the `--no-hashes` option of the `uv export` command.


Then, I needed to use `uv` to generate the manifest too. And `uv` has this nice feature where a tool can be invoked without installing it, which is handy for the `rsconnect-python` package. Here the `--entrypoint` needs to be set up so that Posit Connect knows that the app is in the installed package.

The full `uv` set of commands is:

```
$ uv sync #to update the project environment
$ uv export --no-hashes -o requirements.txt #to generate the the requirements.txt file
$ uvx --from rsconnect-python --python .venv/bin/python rsconnect write-manifest shiny .  --entrypoint pyshinywikidata.app:app #to generate the manifest.json file. note the entrypoint.
```

At this point `git status` said I have new files in the repository, as expected. 
So then I just added them to the repository.

## Step 5 in which I decide to follow a friends advice and move the generation of `requirements.txt` and `manifest.json` to GitHub actions

Whenever changes to the code are made `requirements.txt` and `manifest.json` may need to be 
regenerated and committed to the repository so that Posit Connect knows how to update the app.
But, forgetting to do this would not be strange. So why not automate it?

Posit Connect can only listen to branches, so the idea is to have a `deploy` branch which Connect publishes,
but which is managed by GitHub actions.

With a little help from existing `yaml` files I stitched together a file that creates the needed files on the deploy branch and then successfully deployed it to Posit Connect.

Amazing!

Additionally, I needed to allow workflow permission in my repository settings to be able to read and write. That's under Settings -> Select Actions → General -> Workflow -> Read and write permissions.

All of the code and my trials and errors are under the repo at: [https://github.com/novica/pyshinywikidata/](https://github.com/novica/pyshinywikidata/).

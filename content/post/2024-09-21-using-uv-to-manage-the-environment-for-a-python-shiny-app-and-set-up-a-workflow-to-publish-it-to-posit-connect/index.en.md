---
title: Using uv to manage the environment for a Python Shiny app and setting up a GitHub action 
  to publish it to Posit Connect
author: novica
date: '2024-09-21'
slug: using-uv-to-manage-the-environment-for-a-python-shiny-app-and-set-up-a-workflow-to-publish-it-to-posit-connect
categories:
  - python
  - automation
tags:
  - shiny
  - uv
  - github actions
subtitle: ''
summary: 'A step-by-step guide to work with a Python Shiny app with the help of 
uv and a GitHub Actions workflow to have it published from GitHub to Posit Connect.'
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

A couple of weeks ago, I got a notification from LinkedIn. Unlike the usual notifications, 
this was not from an anonymous recruiter viewing my profile. It was a post by 
[Russ Hyde](https://www.jumpingrivers.com/authors/russ-hyde/) who was looking for examples on
how to organize the code in a Python Shiny app. He bumped into my repository on the 
[topic](https://github.com/novica/pyshinywikidata). For the curious, there is also an accompanying
[blog post](/post/packaging-a-python-shiny-app/) where I describe a simple approach 
to package a Python Shiny app.

It was a great reminder that I should look into my previous work, as I was 
anyhow trying to figure out different ways to deploy Python Shiny apps to Posit Connect. 

At the same time the relatively new Python packager and project manager called 
[uv](https://docs.astral.sh/uv/) caught my attention with many internet resources
about its features popping up, more or less, at the same time (for example: 
[demo project by Damjan](https://github.com/gdamjan/uv-getting-started), 
[Unified Python packaging with uv](https://talkpython.fm/episodes/show/476/unified-python-packaging-with-uv)). 

So I thought why not test how Shiny for Python would
work with `uv` and whether this package manager can be used to in a setup
for deployments to Posit Connect.

## Step 1: Start a new `uv` project and add your code

Setting up a new `uv` project is pretty straightforward, and their 
[projects guide](https://docs.astral.sh/uv/guides/projects/) make is even 
simpler.

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
everything (including the `.git` folder and `.gitignore` file) to the `uv` 
project folder, and removed the `hello.py`. Now the Shiny Python application is
part of the `uv` directory.

## Step 2: Project-specific python version

Next, I needed a specific version of Python (my Posit Connect instance runs on 3.11.5).

First, I updated my `pyproject.toml` to have the required Python version.

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

## Step 3: Install Shiny and other dependencies for the app

The example application depends on three packages: `shiny`, `SPARQLWrapper`, 
and `pandas`, and I added those. Adding this to the `uv` environment can be 
done with `uv add`:

```
(py-shiny-uv) $ uv add pandas shiny SPARQLWrapper
```

Then I just verified that the packages load by running python and importing.

## Step 4: Create the `requirements.txt` and `manifest.json` needed for deploying to Posit Connect

As you might know if you are working with Shiny app in Python, the 
`rsconnect-python` package is needed to generate the `manifest.json` file. The
manifest is used by Posit Connect when publishing from a git repository
(which is something that I want to do).

Usually the way to do it is to run in the app folder:

```
$ rsconnect write-manifest shiny .
```

But since the idea is to use `uv` I had to try, and fail multiple times, with it.

First, the problem with `rsconnect` is that it generates the files in the 
app directory, instead of the top level python project. Moving the files is a 
possibility, of course, but it seems it is an unnecessary complication.

The default way to get the requirements with `uv` is:

```
$ uv export -o requirements.txt
```

This, however, generates the dependencies with hashes, which then is a problem 
with the package environment not having a hash. To quote the error log:


> The editable requirement pyshinywikidata cannot be installed when requiring hashes, because there is no single file to hash.


Omitting the package with:

```
$ uv export --no-emit-project -o requirements.txt
```

Fails because now the package containing the app is no longer in 
`requirements.txt`, and Posit Connect can't find the module to run.

Finally, after a few more trial and errors, I found the solution in the 
`--no-hashes` option of the `uv export` command.

Then, I needed to use `uv` to generate the manifest too. And `uv` has this nice 
feature where a tool can be invoked without installing it, which is handy for 
the `rsconnect-python` package. Here, the `--entrypoint` needs to be set up so 
that Posit Connect knows that the app is in the installed package.

The full `uv` set of commands is:

```
# update the project environment
$ uv sync 

# generate the the requirements.txt file
$ uv export --no-hashes -o requirements.txt 

# generate the manifest.json file. note the entrypoint.
$ uvx --from rsconnect-python --python .venv/bin/python rsconnect write-manifest shiny .  --entrypoint pyshinywikidata.app:app 
```

At this point `git status` said I have new files in the repository, as expected.
So then I just added them to the repository.

## Step 5: Generate `requirements.txt` and `manifest.json` with GitHub actions

Whenever changes to the code are made, `requirements.txt` and `manifest.json` 
may need to be regenerated and committed to the repository so that 
Posit Connect knows how to update the app. But, forgetting to do this would not 
be strange. So why not automate it?

Posit Connect can only listen to branches, so the idea is to have a `deploy` 
branch which Connect publishes, but which is managed by GitHub actions.

With a little help from existing `yaml` files, I stitched together a 
[workflow script](https://github.com/novica/pyshinywikidata/blob/main/.github/workflows/update-requirements.yaml) 
that creates the needed files on the `deploy` branch and then successfully deployed it 
to Posit Connect.

Amazing!

Additionally, I needed to allow workflow permission in my repository settings 
to be able to read and write. That's under 
`Settings -> Select Actions → General -> Workflow -> Read` and write permissions.

All the code and my trials and errors are under the repo at: 
[https://github.com/novica/pyshinywikidata/](https://github.com/novica/pyshinywikidata/).

## Summary 
In this article, I reviewed the procedure of setting up a `uv` project manager
environment for a Python Shiny application and integrating the project with 
GitHub Actions to enable automated deployment to Posit Connect. 

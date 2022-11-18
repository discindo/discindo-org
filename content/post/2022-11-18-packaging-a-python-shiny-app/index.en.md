---
title: Packaging a Python Shiny app
author: novica
date: '2022-11-18'
slug: packaging-a-python-shiny-app
categories:
  - Python
tags:
  - shiny
  - package
  - golem
subtitle: ''
summary: ''
authors: []
lastmod: '2022-11-18T21:52:39+01:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

After spending few years writing R Shiny apps with {golem}, the first thing I thought about when writing the Python Shiny [demo](https://github.com/novica/pyshinywikidata) was is it possible to convert it to a python package and if yes how to it.

I was surprised to learn that it is actually pretty simple to do that, especially when compared to the {golem} environment, which I feel has a learning curve to master.

The template app for Shiny for Python is generated with:

```
shiny create .
```

Which produces a really simple directory structure

```
shiny_app
├── app.py
```


To get to a python package we just need to move some things around and create this structure:

```
shiny_app/
├── LICENSE
├── pyproject.toml
├── README.md
├── src/
│   └── shiny_app/
│       ├── __init__.py
│       └── app.py
└── tests/
```

Top to bottom, most of the things are self-explanatory: a license file and a readme, and a directory for `tests`. The `src` folder is where the main code lives, which is the equivalent of the `R` directory in `R` packages and in `{golem}`.

`__init__.py` is a python specific file, required to import the directory as a package. It should be empty.

And, `pyproject.toml` is the equivalent of a `DESCRIPTION` file in `R`. A sample `toml` file is included in the Packaging Python Project [tutorial](https://packaging.python.org/en/latest/tutorials/packaging-projects/).

An app though is rarely one file, so in the code folder additional `py` files may live, like files with functions (`fct` files in {golem}) or with shiny modules (`mod` files in {golem}). The nice thing about python though, is that these can live in their own sub folders. A big and complicated shiny app may look like this when packaged.


```
shiny_app/
├── LICENSE
├── pyproject.toml
├── README.md
├── src/
│   └── shiny_app/
│       ├── __init__.py
│       ├── app.py
│       └── helpers.py
|       └── fct/
│           ├── __init__.py
│           ├── fct_1.py
│           ├── fct_2.py
│           └── fct_3.py
|       └── mod/
│           ├── __init__.py
│           ├── mod_1.py
│           ├── mod_2.py
│           └── mod_3.py
└── tests/
```

To make it work, the imports in the main `app.py` file should include the folder names as well as the module name and functions needed.

```
from .helpers import helpe1, helper2
from .fct.fct_1 import function_1
from .mod/mod_1.py import modUI, modServer
```

Once everything is ready, the package can be build with running the following in the folder where the `toml` file is:

```
python3 -m build
```

Then installed with:

```
pip install .
```

Finally ran with:

```
uvicorn shiny_app.app:app --host 127.0.0.1 --port 8000
```
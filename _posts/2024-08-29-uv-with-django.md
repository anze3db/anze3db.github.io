---
title: "UV with Django"
description: "Using UV to manage dependencies of your Django application."
date: 2024-08-29 00:00:00 +0000
image: assets/cards/2024-08-29-uv-with-django.png
---

[Astral](https://astral.sh/) made a huge summer splash in the Python community last week when they released [`uv` 0.3.0](https://astral.sh/blog/uv-unified-python-packaging). 

<!-- One could say that the uv values are very high this summer ☀️ -->

## What is `uv`?

`uv` is a Python package manager written in rust that has just gained the ability to be a project management tool (like [Pipenv](https://pipenv.pypa.io/en/latest/)/[PDM](https://pdm-project.org/en/latest/)), tool management ([pipx](https://github.com/pypa/pipx)), python installer ([pyenv](https://github.com/pyenv/pyenv)), and more!

I was very eager to try it out on my Django projects, but the init rial 0.3.0 release was designed for managing installable Python packages, so there were [a few rough edges](https://github.com/astral-sh/uv/issues/6321) when using it to manage Django app dependencies. Only a week later, these issues have been addressed, defaults were switched around, and [`uv 0.4.0` now supports Python projects](https://github.com/astral-sh/uv/releases/tag/0.4.0) like your Django application out of the box!

## Using `uv` to create a new Django project

First, we'll need to get `uv`. See [the official docs](https://docs.astral.sh/uv/getting-started/installation/) for all the installation options, but the easiest way to get it on Linux and MacOS is to run:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Now that we have `uv`, we can use it to start a new project with:

```bash
❯ uv init hello-django
Initialized project `hello-django` at `/Users/anze/Coding/hello-Django`
```

<mark>Make sure you are using uv <strong>0.4.0</strong> or newer; older versions will presume you are creating an installable Python package, and you'll see "error: Failed to prepare distributions" errors when trying to run your project. You can upgrade your uv version by running the install command above.</mark>

You can now `cd` into the `hello-django` folder and see that `uv` created three files for us: 

```
README.md
hello.py # we won't need this so feel free to rm it.
pyproject.toml
```

The `pyproject.toml` file is the most interesting one since it defines two important properties: `requires-python` and `dependencies`. The former defines which Python version we will be using, and the latter defines the project dependencies.

```toml
[project]
name = "hello-django"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = []
```

Our Django projects won't really care about the `name`, `version`, `description`, and `readme` properties, so just leave them as is.

Speaking of project dependencies. It's about time we install Django with `uv add django`!

```bash
❯ uv add django
Using Python 3.12.5
Creating virtualenv at: .venv
Resolved 5 packages in 186ms
Prepared 3 packages in 3ms
Installed 3 packages in 235ms
 + asgiref==3.8.1
 + django==5.1
 + sqlparse==0.5.1
```

We can see that uv created a virtual environment (`.venv` folder), and installed Django with its two dependencies (`asgiref` and `sqlparse`). If we inspect the `pyproject.toml` file, we can see that Django was added to the dependencies list:

```toml
...
dependencies = [
    "django>=5.1",
]
```

The Django version has no upper bonds, so we can easily upgrade it when newer versions of Django come out (using `uv lock --upgrade`). 

This doesn't mean our current dependencies aren't locked tight. Our whole dependency tree has its versions specified in the `uv.lock` file. The lock file is a [cross-platform lock file](https://docs.astral.sh/uv/concepts/projects/#project-lockfile), so it should be safe to install on any operating system!

Now that we have Django installed, we can run Django's `startproject` command:

```bash
uv run django-admin startproject hello .
```

This initialized our Django project, including the `manage.py` file, `hello/settings.py`, etc.

We can start the Django development server with the following:

```bash
uv run manage.py runserver
```

## Using `uv` with an existing Django project

If you already have a Django project, you can still use the `uv init` command to switch to uv.

```bash
cd to_existing_project
❯ uv init .
Initialized project `hello-django` at `/Users/anze/Coding/blog/hello-django`
```

If your project already has a pyproject.toml file defined the command might fail with:

```bash
error: Project is already initialized in `/Users/anze/Coding/blog/hello-django`
```

In that case, you'll need to add a project table to your pyproject.toml file:

```toml
[project]
name = "hello-django"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = []
```

And now, you should be able to add your existing dependencies to the pyproject.toml, either manually or with the `uv add` command. After all the dependencies are specified in `pyproject.toml` you can run `uv sync` to make sure everything is installed in your environment.

## Installing dev dependencies

`uv` also supports installing development dependencies:

```bash
❯ uv add --dev pytest pytest-django
Resolved 11 packages in 8ms
Prepared 5 packages in 0.91ms
Installed 5 packages in 12ms
 + iniconfig==2.0.0
 + packaging==24.1
 + pluggy==1.5.0
 + pytest==8.3.2
 + pytest-django==4.8.0
```

You can now run your tests with

```bash
uv run pytest
```

The dev dependencies are saved in the `tool.uv.dev-dependencies` list in your `pyproject.toml`:

```toml
[tool.uv]
dev-dependencies = [
    "pytest>=8.3.2",
    "pytest-django>=4.8.0",
]
```

When deploying your Django application to production, you can avoid installing dev dependencies by running:

```bash
❯ uv sync --no-dev
Resolved 11 packages in 2ms
Uninstalled 5 packages in 35ms
 - iniconfig==2.0.0
 - packaging==24.1
 - pluggy==1.5.0
 - pytest==8.3.2
 - pytest-django==4.8.0
```

## Avoiding uv run

Writing `uv run` gets very old very fast, but there are a few options to make the experience a bit nicer.

### 1. Aliases

You can alias `uv run` to something shorter like `uvr`:

```bash
alias uvr="uv run"
```

Or for Django use cases, you can define `uvm` like so:

```bash
alias uvm="uv run python manage.py"
```
So that you can run the `manage.py` file with only three letters:

```bash
uvm runserver
```

### 2. Adjusting the [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line in `manage.py`

You can change the `#!/usr/bin/env python` at the top of `manage.py` into `#!/usr/bin/env -S uv run` to force invocations to use `uv run`.

```bash
./manage.py runserver
```

I learned about this trick from Jeff Triplett's blog [Python UV run with shebangs](https://micro.webology.dev/). 💚

## Using `uv run --with`

`uv run` has another trick up its sleeve: an optional `--with` parameter that allows you to run your project with a different package version. This is super useful if you want to quickly verify if your project works with a newer (or older) version of Django:

```bash
❯ uv run python manage.py version
5.1
❯ uv run --with 'django<5' manage.py version
4.2.15
❯ uv run --with 'django<5' pytest # to run your tests on the latest 4.x version
```

## Fin

This is a really exciting time for Python and Python packaging! If you want to learn more about `uv` check out the following links:

 * [The official documentation](https://docs.astral.sh/uv/)
 * Simon Willison wrote some [notes on uv](https://simonwillison.net/2024/Aug/20/uv-unified-python-packaging/)
 * Jeff Triplet also wrote about the [uv updates](https://micro.webology.dev/2024/08/21/uv-updates-and.html)

If you have any other tricks or ideas to simplify your Django workflows, [let me know](mailto:anze@pecar.me), and I'll add your suggestions to the blog post!
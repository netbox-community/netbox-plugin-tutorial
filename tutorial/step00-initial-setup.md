# Step 0: Initial Setup

Before we start writing our plugin, we need a working local NetBox development environment. In this step we'll:

- Install (or verify) a local NetBox dev setup
- Enable development-friendly settings
- Optionally clone the demo plugin repository so you can follow along (or jump between tutorial steps)

:warning: **Warning:** This tutorial requires **NetBox v4.5 or later**. Attempting to use an earlier NetBox release will not work.

## Install the NetBox Development Environment

Plugin development requires a local installation of NetBox.

If you don't already have NetBox installed, follow the official [installation instructions](https://netbox.readthedocs.io/en/stable/installation/). For development, installing from the Git repository is recommended because it makes it easy to switch between NetBox versions/tags.

:green_circle: **Tip:** If you're setting this up *only* for local development, you can usually stop once you can successfully start the development server (e.g. `manage.py runserver`). You can always come back later and complete any optional production hardening steps.

Before you start developing, make sure NetBox is running in development mode with debugging enabled. From your NetBox installation root, edit `netbox/netbox/configuration.py` and set:

```python
DEBUG = True
DEVELOPER = True
```

These settings ensure that:

- static assets can be served by the development server
- you'll get full tracebacks in the browser when an error occurs

:blue_square: **Note:** If NetBox is already running, restart the development server after changing these settings.

:warning: **Warning:** Never enable `DEBUG = True` in a production environment.

## Prepare the Plugin Development Environment

A NetBox plugin is a standard Python package that contains a NetBox-specific Django application. Your plugin project should be structured according to the [Python Packaging User Guide](https://packaging.python.org/en/latest/tutorials/packaging-projects/).

There are two ways to follow this tutorial:

1. **Create the directory structure manually** (you can optionally use the demo repository as a reference).
2. **Use the cookiecutter template** ([cookiecutter-netbox-plugin](https://github.com/netbox-community/cookiecutter-netbox-plugin)) to generate a new plugin project.

In this tutorial, we'll use the **manual approach**, building up a project that will eventually look similar to this:

```text
.
├── netbox_access_lists
│   ├── api
│   │   ├── __init__.py
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   └── views.py
│   ├── choices.py
│   ├── filtersets.py
│   ├── forms.py
│   ├── graphql
│   │   ├── enums.py
│   │   ├── filters.py
│   │   ├── __init__.py
│   │   ├── schema.py
│   │   └── types.py
│   ├── __init__.py
│   ├── migrations
│   │   ├── 0001_initial.py
│   │   └── __init__.py
│   ├── models.py
│   ├── navigation.py
│   ├── search.py
│   ├── tables.py
│   ├── templates
│   │   └── netbox_access_lists
│   │       ├── accesslist.html
│   │       └── accesslistrule.html
│   ├── urls.py
│   └── views.py
├── pyproject.toml
└── README.md
```

:blue_square: **Note:** You don't need to create all of this up front. We'll add files and directories as we go.

If you prefer to use the cookiecutter template, you can skip ahead to [Step 1](/tutorial/step01-plugin-definition.md). Keep in mind that cookiecutter generates a slightly different (and more complete) project layout, which is especially helpful if you plan to publish your plugin on PyPI.

If you're following the manual approach and want a reference implementation to compare against, see the next section.

### Demo Git repository (optional)

The demo repository contains the completed plugin project from this tutorial. Each tutorial step builds on the previous one, and the demo repo includes a snapshot of the code at each step as a separate Git branch.

To clone the demo repository, `cd` to your preferred workspace (your home directory is fine) and run:

```bash
git clone --branch step00-empty https://github.com/netbox-community/netbox-plugin-demo
```

:blue_square: **Note:** Cloning the demo repository is optional, but it makes it easy to jump between steps and compare your work to a known-good implementation if you get stuck.

This completes our initial setup.

<div align="center">

[Step 1: Plugin Definition](/tutorial/step01-plugin-definition.md) :arrow_right:

</div>

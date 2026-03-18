# Step 1: Plugin Definition

Make sure you've completed the [initial setup](/tutorial/step00-initial-setup.md) before proceeding.

Our plugin will provide a simple interface for managing access lists in NetBox.
We'll call it `netbox_access_lists`, and it will live under `/plugins/access-lists/` in the web UI.

In this step, we'll create the minimal skeleton NetBox needs to recognize our plugin:

- a Python package (`netbox_access_lists/`)
- a `PluginConfig` definition in `__init__.py`
- basic project metadata (`pyproject.toml`) and docs (`README.md`)
- an editable install into NetBox's virtual environment

By the end, NetBox should list the plugin under **Admin > System > Plugins**. (It won't do much yet.)

## Plugin Configuration

Before we can begin developing our plugin, navigate to the project root directory.

If you cloned the demo repository, you can directly `cd` into it:

```bash
cd netbox-plugin-demo
```

Otherwise, create a new directory for your plugin's code and then `cd` into it:

```bash
mkdir -p netbox-plugin-demo
cd netbox-plugin-demo
```

:blue_square: **Note:** The name of this directory doesn't matter. Many developers use the distribution name with hyphens (e.g. `netbox-access-lists`). To keep the tutorial consistent (and less confusing), we'll use `netbox-plugin-demo` as the project root directory.

### Create package root

Our plugin is for managing access lists in NetBox, so we'll give the Python package an appropriate name, such as `netbox_access_lists`.

:blue_square: **Note:** It's recommended to prefix the name of your plugin with `netbox_` to reduce the chance of collisions.

First, create a **directory** to hold our plugin's Python code (i.e., the plugin's package root) named `netbox_access_lists`:

```bash
mkdir -p netbox_access_lists
```

Mind the difference between our *project* root directory and the plugin's *package* root:

```text
netbox-plugin-demo/        # project root
└── netbox_access_lists/   # package root (Python package)
```

:warning: **Warning:** The name of the plugin directory must match the name of the Python package it defines, and it must use **underscores** (not hyphens).

### Create the `PluginConfig` class

The `PluginConfig` class holds all the information NetBox needs to load our plugin.

In the package root, create a new file named `__init__.py`:

```bash
touch netbox_access_lists/__init__.py
```

Open `__init__.py` in the text editor of your choice and import `PluginConfig` at the top of the file:

```python
from netbox.plugins import PluginConfig
```

Next, create a new class named `NetBoxAccessListsConfig` by subclassing `PluginConfig`.
There are [many optional attributes](https://netboxlabs.com/docs/netbox/plugins/development/#pluginconfig-attributes) that can be set here, but for now we only need a few:

```python
class NetBoxAccessListsConfig(PluginConfig):
    name = 'netbox_access_lists'
    verbose_name = 'NetBox Access Lists'
    description = 'Manage simple access lists in NetBox'
    version = '0.1.0'
    base_url = 'access-lists'
    min_version = '4.5.0'
    max_version = '4.5.99'
```

:blue_square: **Note:** `min_version` and `max_version` let you declare which NetBox versions your plugin supports. It's common to pin to a NetBox minor release while developing, then widen the range once you've validated compatibility.

Finally, we need to expose this class as `config` so that NetBox can detect it.
Add this line to the end of the file:

```python
config = NetBoxAccessListsConfig
```

## Create a README

It's considered best practice to include a `README.md` file with any project you publish.
This is a brief piece of documentation that explains what your project does, how to install/run it, where to find help, etc.

Because this is just a learning exercise, we have little to say about our plugin, but go ahead and create the file anyway.

Back in the project root (one level up from `__init__.py`), create a file named `README.md`:

```bash
touch README.md
```

Open the file in your text editor and enter the following:

```markdown
# NetBox Access Lists

Manage simple access lists in NetBox.
```

:green_circle: **Tip:** The `.md` extension tells tools (including GitHub) to render the file as Markdown for better readability.

## Install the plugin

### Create `pyproject.toml`

To enable installation of our plugin into the virtual environment we created during NetBox installation, we'll create a simple Python package configuration file.

In the project root directory (where `README.md` is located), create a file named `pyproject.toml` and enter the configuration below.
Ensure that the `name` attribute matches the name of the plugin as it would be listed on PyPI (i.e. `netbox-access-lists`).

```bash
touch pyproject.toml
```

Remember to update your contact information as needed.

```toml
# See PEP 518 for the spec of this file
# https://www.python.org/dev/peps/pep-0518/

[build-system]
requires = ["setuptools >= 77.0.3"]
build-backend = "setuptools.build_meta"

[project]
# This is the *distribution* name (e.g., what you'd publish on PyPI), so hyphens are typical.
name = "netbox-access-lists"
version = "0.1.0"
requires-python = ">=3.12.0"
authors = [
    {name = "Tutorial Author", email = "your@email.com"},
]
description = "Manage simple access lists in NetBox"
readme = { file = "README.md", content-type = "text/markdown" }
classifiers = [
    "Development Status :: 3 - Alpha",
    "Framework :: Django",
    "Framework :: Django :: 5.2",
    "Intended Audience :: Developers",
    "Intended Audience :: System Administrators",
    "Natural Language :: English",
    "Operating System :: OS Independent",
    "Programming Language :: Python",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3 :: Only",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
    "Programming Language :: Python :: 3.14",
]

[tool.setuptools]
include-package-data = true

[tool.setuptools.packages.find]
# Include the root package *and* any subpackages we add later (api/, graphql/, etc.).
include = ["netbox_access_lists*"]

[tool.setuptools.package-data]
netbox_access_lists = [
    "templates/**",
    "static/**",
    "locale/**/*.mo"
]
```

:warning: **Warning:** Be sure to create `pyproject.toml` in the project root and _not_ within the `netbox_access_lists` directory.

You should now have a file structure similar to the following:

```text
.
├── netbox_access_lists/
│   └── __init__.py
├── pyproject.toml
└── README.md
```

This file will be used by `pip` to install our plugin into the virtual environment.
There are many other options that can be configured here, but for now we'll stick with the basics.

:green_circle: **Tip:** There are alternative methods for installing Python code that work just as well; feel free to use your preferred approach. Just be aware that this guide assumes the use of `pip` and adjust accordingly.

### Activate the virtual environment

To ensure our plugin is accessible to the NetBox installation, we first need to activate the Python [virtual environment](https://docs.python.org/3/library/venv.html) that was created when we installed NetBox.

If you used the documentation defaults, the venv path will be `/opt/netbox/venv/`:

```bash
source /opt/netbox/venv/bin/activate
```

### Run `pip install`

We can now install our plugin by running `pip install -e .` from the project root.

The `-e` (`--editable`) argument tells `pip` to create a link to our local development directory instead of copying files into the virtual environment.
This avoids the need to re-install the plugin every time we make a change.

```bash
python -m pip install -e .
```

## Configure NetBox

Finally, we need to configure NetBox to enable our new plugin.

Over in the NetBox installation path, open `netbox/netbox/configuration.py` and look for the `PLUGINS` parameter (it should be an empty list).
If it's not yet defined, go ahead and create it.

Add the name of our plugin to this list:

```python
# configuration.py
PLUGINS = [
    'netbox_access_lists',
]
```

:blue_square: **Note:** This uses the **Python package name** (underscores), not the distribution name from `pyproject.toml` (hyphens).

Save the file and start (or restart) the NetBox development server:

```bash
python netbox/manage.py runserver
```

You should see the development server start successfully.
Open NetBox in a new browser window, log in as a superuser, and navigate to the admin UI.
Under **Admin > System > Plugins** you should see our plugin listed.

![NetBox UI: Plugins list](/images/step01-netbox-plugin-list.png)

:green_circle: **Tip:** If you cloned the demo repository, each tutorial step has a corresponding branch you can diff against. For this step, run:
```bash
git diff remotes/origin/step01-plugin-config
```

This completes our plugin definition setup. Now, onto the fun stuff!

<div align="center">

:arrow_left: [Step 0: Initial Setup](/tutorial/step00-initial-setup.md) | [Step 2: Models](/tutorial/step02-models.md) :arrow_right:

</div>

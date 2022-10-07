# Step 1: Initial Setup

Before we can begin work on our plugin, we must first ensure that we have a suitable development environment in place.

## Set Up the Development Environment

### Install NetBox

Plugin development requires a local installation of NetBox. If you don't already have NetBox installed, please consult the [installation instructions](https://netbox.readthedocs.io/en/stable/installation/).

Be sure to enable debugging in your NetBox configuration by setting `DEBUG = True`. This will ensure that static assets can be served by the development server, and return complete tracebacks whenever there's a server error.

:green_circle: **Tip:** If this installation will be for development use only, it is generally necessary to complete only up to step three in the installation guide, culminating with successfully running the NetBox development server (`manage.py runserver`).

:warning: **Warning:** This guide requires NetBox v3.2 or later. Attempting to use an earlier NetBox release will not work.

### Clone the git Repository

Next, we'll clone the demo git repository from GitHub. First, `cd` into your preferred location (your home directory is probably fine), then clone the repo with `git clone`. We're checking out the `init` branch, which will provide us with an empty workspace to start.

```bash
$ git clone --branch step00-empty https://github.com/netbox-community/netbox-plugin-demo
Cloning into 'netbox-plugin-demo'...
remote: Enumerating objects: 58, done.
remote: Counting objects: 100% (58/58), done.
remote: Compressing objects: 100% (42/42), done.
remote: Total 58 (delta 12), reused 58 (delta 12), pack-reused 0
Unpacking objects: 100% (58/58), done.
```

:blue_square: **Note:** It isn't strictly required to clone the demo repository, but it will enable you to conveniently check out snapshots of the code as the lessons progress and overcome any hiccups.

## Plugin Configuration

### Create `__init__.py`

The `PluginConfig` class holds all the information needs to know about our plugin to install it. First, we'll create a subdirectory to hold out plugin's Python code, as well as an `__init__.py` file to hold the `PluginConfig` definition.

```bash
$ mkdir netbox_access_lists
$ touch netbox_access_lists/__init__.py
```

Next, open `__init__.py` in the text editor of your choice and import the `PluginConfig` class from NetBox at the top of the file.

```python
from extras.plugins import PluginConfig
```

### Create the PluginConfig Class

We'll create a new class named `NetBoxAccessListsConfig` by subclassing `PluginConfig`. This will define all the necessary parameters that control the configuration of our plugin once installed. There are [many optional attributes](https://netbox.readthedocs.io/en/stable/plugins/development/#pluginconfig-attributes) that can be set here, but for now we only need to define a few.

```python
class NetBoxAccessListsConfig(PluginConfig):
    name = 'netbox_access_lists'
    verbose_name = ' NetBox Access Lists'
    description = 'Manage simple ACLs in NetBox'
    version = '0.1'
    base_url = 'access-lists'
```

This will be sufficient to install our plugin in NetBox later on. Finally, we need to expose this class as `config` to ensure that NetBox detects it. Add this line to the end of the file:

```python
config = NetBoxAccessListsConfig
```

## Create a README

It's considered best practice to always include a `README` file with any code you publish. This is a brief piece of documentation that explains your project's purpose, how to install/run it, where to find help, etc. Because this is just a learning exercise, we don't have much to say about our plugin, but go ahead and create the file anyway.

Back in the project's root (one level up from `__init__.py`), create a file named `README.md` and enter the following content:

```markdown
## netbox-access-lists

Manage simple access control lists in NetBox
```

:green_circle: **Tip:** You'll notice that we've given our `README` file a `md` extension. This tells tools which support it to render the file as Markdown for better readability.

## Install the Plugin

### Create `setup.py`

To enable the installation of our plugin into the virtual environment we created above, we'll create a simple Python setup script. In the project's root directory, create a file named `setup.py` and enter the code below.

```python
from setuptools import find_packages, setup

setup(
    name='netbox-access-lists',
    version='0.1',
    description='An example NetBox plugin',
    install_requires=[],
    packages=find_packages(),
    include_package_data=True,
    zip_safe=False,
)
```

:warning: **Warning:** Be sure to create `setup.py` in the project root and _not_ within the `netbox_access_lists` directory.

This file will call the `setup()` function provided by Python's [`setuptools`](https://packaging.python.org/en/latest/guides/distributing-packages-using-setuptools/) library to install our code. There are plenty of additional arguments that can be passed, but for our example this is sufficient.

:green_circle: **Tip:** There are alternative methods for installing Python code which work just as well; feel free to use your preferred approach. Just be aware that this guide assumes the use of `setuptools` and adjust accordingly.

### Activate the Virtual Environment

To ensure our plugin is accessible to the NetBox installation, we first need to activate the Python [virtual environment](https://docs.python.org/3/library/venv.html) that was created when we installed NetBox. To do this, determine the virtual environment's path (this will be `/opt/netbox/venv/` if you use the documentation's defaults) and activate it:

```bash
$ source /opt/netbox/venv/bin/activate
```

### Run `setup.py`

We can now install our plugin by running `setup.py`. First, make sure the virtual environment is still active, then run the following command from the project's root. The `develop` argument tells `setuptools` to create a link to our local development path instead of copying files into the virtual environment. This avoids the need to re-install the plugin every time we make a change.

```bash
$ python3 setup.py develop
running develop
running egg_info
creating netbox_access_lists.egg-info
writing manifest file 'netbox_access_lists.egg-info/SOURCES.txt'
writing manifest file 'netbox_access_lists.egg-info/SOURCES.txt'
running build_ext
```

### Configure NetBox

Finally, we need to configure NetBox to enable our new plugin. Over in the NetBox installation path, open `netbox/netbox/configuration.py` and look for the `PLUGINS` parameter; this should be an empty list. (If it's not yet defined, go ahead and create it.) Add the name of our plugin to this list:

```python
# configuration.py
PLUGINS = [
    'netbox_access_lists',
]
```

Save the file and run the NetBox development server (if not already running):

```bash
$ python netbox/manage.py runserver
```

You should see the development server start successfully. Open NetBox in a new browser window, log in as a superuser, and navigate to the admin UI. Under **System > Installed Plugins** you should see our plugin listed.

![Django admin UI: Plugins list](/images/step01-django-admin-plugins.png)

:green_circle: **Tip:** You can check your work at the end of each step in the tutorial by running a `git diff` against the corresponding branch. For example, at the end of step one, run `git diff remotes/origin/step01-initial-setup` to compare your work with the completed step. This will help identify any tasks you might have missed.

This completes our initial setup. Now, onto the fun stuff!

<div align="center">

[Step 2: Models](/tutorial/step02-models.md) :arrow_right:

</div>


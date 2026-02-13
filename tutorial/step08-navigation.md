# Step 8: Navigation

So far, we have been entering URLs manually to access our plugin views. That works for development, but it is not very friendly for regular use.
In this step, we will add links to the NetBox navigation menu so users can reach our pages with a click.

:blue_square: **Note:** If you skipped the previous step and you cloned `netbox-plugin-demo`, run `git checkout step07-templates`.

## Add navigation menu items

Begin by creating `navigation.py` in the `netbox_access_lists/` directory of your plugin project root.

```bash
cd netbox_access_lists/
touch navigation.py
```

We will use NetBox's navigation classes to add menu items and optional buttons:

* [`PluginMenuItem`](https://netboxlabs.com/docs/netbox/en/stable/plugins/development/navigation/#menu-items) for menu links
* [`PluginMenu`](https://netboxlabs.com/docs/netbox/en/stable/plugins/development/navigation/#menu-groups) for grouping links under a plugin section
* [`PluginMenuButton`](https://netboxlabs.com/docs/netbox/en/stable/plugins/development/navigation/#menu-buttons) for shortcut buttons (for example, Add)

At the top of the file, import the navigation classes provided by NetBox:

```python
from netbox.plugins import PluginMenu, PluginMenuButton, PluginMenuItem
```

### Create menu items

We will add a link to the list view for each model. This is done by creating a `PluginMenuItem` with (at minimum) two arguments:

* `link` is the name of the URL we want to link to
* `link_text` is the text shown in the menu

Create two `PluginMenuItem` objects and assign them to `accesslist_item` and `accesslistrule_item`:

```python
# Access List
accesslist_item = PluginMenuItem(
    link='plugins:netbox_access_lists:accesslist_list',
    link_text='Access Lists',
)

# Access List Rule
accesslistrule_item = PluginMenuItem(
    link='plugins:netbox_access_lists:accesslistrule_list',
    link_text='Access List Rules',
)
```

### Group menu items

Next, we will group our items under a single plugin menu section. We will:

* label the plugin menu `Access Lists`
* create two groups, `Access Lists` and `Rules`
* use a lock icon for the menu section

Add this to the end of `navigation.py`:

```python
menu = PluginMenu(
    label='Access Lists',
    groups=(
        (
            'Access Lists',
            (accesslist_item,),
        ),
        (
            'Rules',
            (accesslistrule_item,),
        ),
    ),
    icon_class='mdi mdi-lock',
)
```

When you reload NetBox, you should see a new `Access Lists` section appear in the navigation menu with the two links.

:blue_square: **Note:** If the menu items do not appear, restart the development server (`manage.py runserver`) and refresh the page.

![Navigation menu items](/images/step08-menu-items1.png)

## Add menu buttons

We can also add buttons to the menu items, which is handy for quick access to add forms.

A `PluginMenuButton` supports:

* `link` is the URL name the button should open
* `title` is the tooltip text shown on hover
* `icon_class` controls the icon that is displayed

Create button lists above the `PluginMenuItem` definitions:

```python
accesslist_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslist_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
    )
]

accesslistrule_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslistrule_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
    )
]
```

Then attach them to the menu items using the `buttons` argument:

```python
# Access List
accesslist_item = PluginMenuItem(
    link='plugins:netbox_access_lists:accesslist_list',
    link_text='Access Lists',
    buttons=accesslist_buttons,
)

# Access List Rule
accesslistrule_item = PluginMenuItem(
    link='plugins:netbox_access_lists:accesslistrule_list',
    link_text='Access List Rules',
    buttons=accesslistrule_buttons,
)
```

Now you should see a plus icon appear next to each menu item while hovering.

:blue_square: **Note:** Menu buttons are only shown while hovering over a menu item.

![Navigation menu items with buttons](/images/step08-menu-items2.png)

## Add permission constraints

Although permissions are not fully covered in this tutorial, we can still add basic permission constraints so menu items and buttons only appear for users who are allowed to use them.

To do this, add the `permissions` argument to both `PluginMenuItem` and `PluginMenuButton`.

Permission strings follow the format `app_label.codename`, for example:

* `netbox_access_lists.view_accesslist`
* `netbox_access_lists.add_accesslist`

### Add permissions to buttons

```python
# Access List
accesslist_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslist_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        permissions=['netbox_access_lists.add_accesslist'],
    )
]

# Access List Rule
accesslistrule_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslistrule_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        permissions=['netbox_access_lists.add_accesslistrule'],
    )
]
```

### Add permissions to menu items

```python
# Access List
accesslist_item = PluginMenuItem(
    link='plugins:netbox_access_lists:accesslist_list',
    link_text='Access Lists',
    permissions=['netbox_access_lists.view_accesslist'],
    buttons=accesslist_buttons,
)

# Access List Rule
accesslistrule_item = PluginMenuItem(
    link='plugins:netbox_access_lists:accesslistrule_list',
    link_text='Access List Rules',
    permissions=['netbox_access_lists.view_accesslistrule'],
    buttons=accesslistrule_buttons,
)
```

## Final `navigation.py`

Your complete `navigation.py` should now look like this:

```python
from netbox.plugins import PluginMenu, PluginMenuButton, PluginMenuItem


#
# Define plugin menu buttons
#


# Access List
accesslist_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslist_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        permissions=['netbox_access_lists.add_accesslist'],
    )
]

# Access List Rule
accesslistrule_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslistrule_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        permissions=['netbox_access_lists.add_accesslistrule'],
    )
]

#
# Define plugin menu items
#

# Access List
accesslist_item = PluginMenuItem(
    link='plugins:netbox_access_lists:accesslist_list',
    link_text='Access Lists',
    permissions=['netbox_access_lists.view_accesslist'],
    buttons=accesslist_buttons,
)

# Access List Rule
accesslistrule_item = PluginMenuItem(
    link='plugins:netbox_access_lists:accesslistrule_list',
    link_text='Access List Rules',
    permissions=['netbox_access_lists.view_accesslistrule'],
    buttons=accesslistrule_buttons,
)

#
# Define plugin menu groups
#

menu = PluginMenu(
    label='Access Lists',
    groups=(
        (
            'Access Lists',
            (accesslist_item,),
        ),
        (
            'Rules',
            (accesslistrule_item,),
        ),
    ),
    icon_class='mdi mdi-lock',
)
```

For reference, your plugin project should now include `navigation.py`:

```text
.
├── netbox_access_lists
│   ├── choices.py
│   ├── forms.py
│   ├── __init__.py
│   ├── migrations
│   │   ├── 0001_initial.py
│   │   └── __init__.py
│   ├── models.py
│   ├── navigation.py
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

<div align="center">

:arrow_left: [Step 7: Templates](/tutorial/step07-templates.md) | [Step 9: Filter Sets](/tutorial/step09-filter-sets.md) :arrow_right:

</div>

# Step 7: Navigation

So far, we've been manually entering URLs to access our plugin's views. This obviously will not suffice for regular use, so let's see about adding some links to NetBox's navigation menu.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step06-templates`.

## Adding Navigation Menu Items

Begin by creating `navigation.py` in the `netbox_access_lists/` directory.

```bash
$ cd netbox_access_lists/
$ edit navigation.py
```

We'll need to import the `PluginMenuItem` class provided by NetBox to add new menu items; do this at the top of the file.

```python
from extras.plugins import PluginMenuItem
```

Next, we'll create a tuple named `menu_items`. This will hold our customized `PluginMenuItem` instances.

```python
menu_items = ()
```

Let's add a link to the list view for each of our models. This is done by instantiating `PluginMenuItem` with (at minimum) two arguments:

* `link` - The name of the URL path to which we're linking
* `link_text` - The text of the link

Create two instances of `PluginMenuItem` within `menu_items`:

```python
menu_items = (
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslist_list',
        link_text='Access Lists'
    ),
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslistrule_list',
        link_text='Access List Rules'
    ),
)
```

Upon reloading the page, you should see a new "Plugins" section appear at the end of the navigation menu, and below it, a section titled "NetBox Access Lists", with our two links. Navigating to either of these links will highlight the corresponding menu item.

:blue_square: **Note:** If the menu items do not appear, try restarting the development server (`manage.py runserver`).

![Navigation menu items](/images/step07-menu-items1.png)

That's much more convenient!

### Adding Menu Buttons

While we're at it, we can add direct links to the "add" views for access lists and rules as buttons. We'll need to import two additional classes at the top of `navigation.py`: `PluginMenuButton` and `ButtonColorChoices`.

```python
from extras.plugins import PluginMenuButton, PluginMenuItem
from utilities.choices import ButtonColorChoices
```

`PluginMenuButton` is used similarly to `PluginMenuItem`: Instantiate it with the necessary keyword arguments to effect a menu button. These arguments are:

* `link` - The name of the URL path to which the button links
* `title` - The text displayed when the user hovers over the button
* `icon_class` - CSS class name(s) indicating the icon to display
* `color` - The button's color (choices are provided by `ButtonColorChoices`)

Create these instances in `navigation.py` _above_ `menu_items`. Because each menu item expects to receive an iterable of button instances, we'll create each of these inside a list.

```python
accesslist_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslist_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        color=ButtonColorChoices.GREEN
    )
]

accesslistrule_butons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslistrule_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        color=ButtonColorChoices.GREEN
    )
]
```

The buttons can then be passed to the menu items via the `buttons` keyword argument:

```python
menu_items = (
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslist_list',
        link_text='Access Lists',
        buttons=accesslist_buttons
    ),
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslistrule_list',
        link_text='Access List Rules',
        buttons=accesslistrule_butons
    ),
)
```

Now we should see green "add" buttons appear next to our menu links.

![Navigation menu items with buttons](/images/step07-menu-items2.png)

<div align="center">

:arrow_left: [Step 6: Templates](/tutorial/step06-templates.md) | [Step 8: Filter Sets](/tutorial/step08-filter-sets.md) :arrow_right:

</div>


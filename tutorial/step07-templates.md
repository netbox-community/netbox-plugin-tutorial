# Step 7: Templates

Templates are responsible for rendering HTML content for NetBox views.
Each template is a file that mixes HTML with template code.
In most plugins, each model will need its own template for the object detail view.
Templates can also be created or customized for other views, but the default templates NetBox provides are usually a great fit.

NetBox uses the [Django Template Language](https://docs.djangoproject.com/en/stable/topics/templates/) (DTL).
It will feel familiar if you have used [Jinja2](https://jinja2docs.readthedocs.io/en/stable/), but there are important differences.
DTL is intentionally more limited in the logic it can run.
For example, you cannot execute arbitrary Python code inside a template.
If you plan to build more complex layouts, it is worth reviewing the Django template documentation first.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step06-urls` (if you cloned the repository `netbox-plugin-demo`).

## Template File Structure

NetBox looks for templates within a `templates/` subdirectory (if it exists) inside your plugin package root (for example `netbox_access_lists/`).
Inside `templates/`, create a subdirectory named after the plugin (for example `netbox_access_lists`):

```bash
cd netbox_access_lists/
mkdir -p templates/netbox_access_lists/
```

Your template files will live in this directory.

NetBox provides default templates for most generic views.
The main exception is `ObjectView` (the single object detail view), so we need to create templates for:

* `AccessListView`
* `AccessListRuleView`

By default, each `ObjectView` subclass looks for a template named after its model.
In our case:

* `AccessListView` looks for `netbox_access_lists/accesslist.html`
* `AccessListRuleView` looks for `netbox_access_lists/accesslistrule.html`

You *can* override this by setting `template_name` on the view, but the default behavior works well for this tutorial.

For reference, your plugin project should now include the templates directory:

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
│   ├── tables.py
│   ├── templates
│   │   └── netbox_access_lists
│   ├── urls.py
│   └── views.py
├── pyproject.toml
└── README.md
```

## Create the AccessList Template

Create the file `accesslist.html` in the plugin template directory:

```bash
touch templates/netbox_access_lists/accesslist.html
```

Even though we are creating our own template, NetBox has already done a lot of work for us.
We can extend the built-in object template. At the top of the file, add:

```django
{% extends 'generic/object.html' %}
{% load helpers %}
```

This tells Django to load NetBox's `generic/object.html` first, then render whatever we define inside `block` tags.

:green_circle: **Tip:** We load `helpers` so we can use NetBox template tags and filters like `badge`, `linkify`, and `placeholder`.

Now add a `content` block that displays some basic information about the access list:

```html
{% block content %}
  <div class="row">
    <div class="col col-md-6">
      <div class="card">
        <h2 class="card-header">Access List</h2>
        <table class="table table-hover attr-table">
          <tr>
            <th scope="row">Name</th>
            <td>{{ object.name }}</td>
          </tr>
          <tr>
            <th scope="row">Default Action</th>
            <td>{% badge object.get_default_action_display bg_color=object.get_default_action_color %}</td>
          </tr>
          <tr>
            <th scope="row">Rule Count</th>
            <td>{{ object.rules.count }}</td>
          </tr>
        </table>
      </div>
      {% include 'inc/panels/custom_fields.html' %}
    </div>
    <div class="col col-md-6">
      {% include 'inc/panels/tags.html' %}
      {% include 'inc/panels/comments.html' %}
    </div>
  </div>
{% endblock content %}
```

This layout uses one Bootstrap row with two columns:

* The left column shows a card with key fields plus the custom fields panel
* The right column shows the tags and comments panels

:green_circle: **Tip:** If you are unsure how to structure a page, NetBox's own core templates are a great source of examples.

Now try it out.
Go back to the list view at <http://localhost:8000/plugins/access-lists/access-lists/>, then click an access list name to open its detail page.
You should see a page similar to the image below.

:blue_square: **Note:** If NetBox still says the template does not exist, restart the development server (`manage.py runserver`) and try again.

![Access list view](/images/step07-accesslist1.png)

This is a solid start, but it would be even more useful if the access list page also showed its rules.

There are two common ways to do this:

* Extend the existing template to include a rules table on the same page
* Add a separate tab for rules on the access list detail view

You can pick either approach.
If you implement both, you will see the rules twice, which is usually not what you want.

### Extend the Existing Template to Include a Rules Table

To render a rules table, we first need to provide additional context data from the view.

Open `views.py` and find the `AccessListView` class. Add a `get_extra_context()` method like this:

```python
@register_model_view(models.AccessList)
class AccessListView(generic.ObjectView):
    queryset = models.AccessList.objects.all()

    def get_extra_context(self, request, instance):
        """Add rules table to access list view context."""
        rules = instance.rules.restrict(request.user, 'view')
        rules_table = tables.AccessListRuleTable(rules)
        rules_table.columns.hide('access_list')  # Hide AccessList column
        rules_table.configure(request)

        return {
            'rules_table': rules_table,
        }
```

This method:

1. Builds a queryset of rules the current user is allowed to view
2. Creates an `AccessListRuleTable` for those rules
3. Hides the `access_list` column since we are already viewing the parent access list
4. Configures the table for the current request so user preferences are respected
5. Exposes the table in the template as `rules_table`

Now update `accesslist.html` so it can render the table.

First, load `render_table` from `django_tables2` by adding this near the top of the file under the existing `{% load helpers %}` line:

```django
{% load render_table from django_tables2 %}
```

Then, add the card below near the end of the `content` block, just before `{% endblock content %}`:

```html
  <div class="row">
    <div class="col col-md-12">
      <div class="card">
        <h5 class="card-header">Rules</h5>
        <div class="table-responsive">
          {% render_table rules_table %}
        </div>
      </div>
    </div>
  </div>
```

Refresh the access list detail page.
You should now see the rules table at the bottom:

![Access list view with rules table](/images/step07-accesslist2.png)

### Add a Separate Tab to the Access List View

If you would rather show rules on their own tab, you can use `ObjectChildrenView`.

First, update your imports in `views.py` to include `ViewTab`. For example:

```python
from utilities.views import ViewTab, register_model_view
```

Then add this view class below `AccessListView`:

```python
@register_model_view(models.AccessList, 'rules')
class AccessListRulesView(generic.ObjectChildrenView):
    queryset = models.AccessList.objects.all()
    child_model = models.AccessListRule
    table = tables.AccessListRuleTable
    tab = ViewTab(
        label='Rules',
        badge=lambda obj: obj.rules.count(),
        permission='netbox_access_lists.view_accesslistrule',
        weight=500,
    )

    def get_children(self, request, parent):
        return parent.rules.restrict(request.user, 'view').all()

    def get_table(self, *args, **kwargs):
        rules_table = super().get_table(*args, **kwargs)
        rules_table.columns.hide('access_list')  # Hide AccessList column
        return rules_table
```

An `ObjectChildrenView` is similar to an `ObjectView`, but it renders a child object table on its own tab. Here:

* `child_model` tells NetBox which related objects to display
* `table` defines how to render those objects
* `tab` controls how the tab appears in the UI (label, optional badge, permission, and ordering)

The `get_children` method is called to retrieve the child objects to display in the tab.
It limits the returned queryset to objects the user has permission to view (`.restrict(request.user, 'view')`).

Reload the access list detail page, and you should see a new Rules tab.
When you click it, you will get the rules table on that tab:

![Access list view with rules tab](/images/step07-accesslist3.png)

## Create the AccessListRule Template

Now we need a template for `AccessListRule` as well.
Create `accesslistrule.html` next to the first template:

```bash
touch templates/netbox_access_lists/accesslistrule.html
```

Add the following content:

```html
{% extends 'generic/object.html' %}
{% load helpers %}

{% block content %}
  <div class="row">
    <div class="col col-md-6">
      <div class="card">
        <h2 class="card-header">Access List Rule</h2>
        <table class="table table-hover attr-table">
          <tr>
            <th scope="row">Access List</th>
            <td>{{ object.access_list|linkify }}</td>
          </tr>
          <tr>
            <th scope="row">Index</th>
            <td>{{ object.index }}</td>
          </tr>
          <tr>
            <th scope="row">Description</th>
            <td>{{ object.description|placeholder }}</td>
          </tr>
        </table>
      </div>
      {% include 'inc/panels/custom_fields.html' %}
      {% include 'inc/panels/tags.html' %}
    </div>
    <div class="col col-md-6">
      <div class="card">
        <h2 class="card-header">Details</h2>
        <table class="table table-hover attr-table">
          <tr>
            <th scope="row">Protocol</th>
            <td>{% badge object.get_protocol_display bg_color=object.get_protocol_color %}</td>
          </tr>
          <tr>
            <th scope="row">Source Prefix</th>
            <td>{{ object.source_prefix|linkify|placeholder }}</td>
          </tr>
          <tr>
            <th scope="row">Source Ports</th>
            <td>
              {% if object.source_ports %}
                {{ object.source_ports|join:", " }}
              {% else %}
                {{ ""|placeholder }}
              {% endif %}
            </td>
          </tr>
          <tr>
            <th scope="row">Destination Prefix</th>
            <td>{{ object.destination_prefix|linkify|placeholder }}</td>
          </tr>
          <tr>
            <th scope="row">Destination Ports</th>
            <td>
              {% if object.destination_ports %}
                {{ object.destination_ports|join:", " }}
              {% else %}
                {{ ""|placeholder }}
              {% endif %}
            </td>
          </tr>
          <tr>
            <th scope="row">Action</th>
            <td>{% badge object.get_action_display bg_color=object.get_action_color %}</td>
          </tr>
        </table>
      </div>
    </div>
  </div>
{% endblock content %}
```

A few details are worth calling out:

* `linkify` turns related objects into links when possible.
* `placeholder` renders a friendly placeholder for empty values.
* For ports, we use an `{% if %}` block so `join` is only called when a list is present.
* `badge` renders colored labels for choice fields. In templates, you reference display helpers without parentheses, for example `object.get_action_display`. This is a [Django convention](https://docs.djangoproject.com/en/stable/ref/models/instances/#extra-instance-methods) for static choice fields to return the human friendly label rather than the raw value. The color is determined by the `get_protocol_color()` and `get_action_color()` methods.

Once you save the file, open a rule detail page in the UI and you should see something like this:

![Access list rule view](/images/step07-accesslistrule.png)

Feel free to experiment with different layouts and content before moving on.

For reference, your plugin project should now look like this:

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

:arrow_left: [Step 6: URLs](/tutorial/step06-urls.md) | [Step 8: Navigation](/tutorial/step08-navigation.md) :arrow_right:

</div>

# Step 6: Templates

Templates are responsible for rendering HTML content for NetBox views. Each template exists as a file with a mix of HTML and template code. Generally speaking, each model in a NetBox plugin must have its own template. Templates may also be created or customized for other views, but the default templates NetBox provides are suitable in most cases.

NetBox's rendering backend uses the [Django Template Language](https://docs.djangoproject.com/en/stable/topics/templates/) (DTL). It will immediately look very familiar if you've used [Jinja2](https://jinja2docs.readthedocs.io/en/stable/), but be aware that there are some important differences between the two. Generally, DTL is much more limited in the types of logic it can execute: Directly executing Python code, for instance, is not possible. Be sure to study the Django documentation before attempting to create any complex templates.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step05-views`.

## Template File Structure

NetBox looks for templates within the `templates/` directory (if it exists) within the plugin root. Within this directory, create a subdirectory bearing the name of the plugin:

```bash
$ cd netbox_access_lists/
$ mkdir -p templates/netbox_access_lists/
```

The template files will reside in this directory. Default templates are provided for all generic views except for `ObjectView`, so we'll need to create templates for our `AccessListView` and `AccessListRuleView` views.

By default, each `ObjectView` subclass will look for a template bearing the name of its associated model. For instance, `AccessListView` will look for `accesslist.html`. This can be overriden by setting `template_name` on the view, but this behavior is suitable for our purposes.

## Create the AccessList Template

Begin by creating the file `accesslist.html` in the plugin's template directory.

```bash
$ edit templates/netbox_access_lists/accesslist.html
```

Although we need to create our own template, NetBox has done much of the work for us, and provides a generic template that we can easily extend. At the top of the file, add an `extends` tag:

```
{% extends 'generic/object.html' %}
```

This tells the rendering engine to first load the NetBox template at `generic/object.html` and populate only the content we provide within `block` tags.

Let's extend the generic template's `content` block with some information about the access list.

```
{% block content %}
  <div class="row mb-3">
    <div class="col col-md-6">
      <div class="card">
        <h5 class="card-header">Access List</h5>
        <div class="card-body">
          <table class="table table-hover attr-table">
            <tr>
              <th scope="row">Name</th>
              <td>{{ object.name }}</td>
            </tr>
            <tr>
              <th scope="row">Default Action</th>
              <td>{{ object.get_default_action_display }}</td>
            </tr>
            <tr>
              <th scope="row">Rules</th>
              <td>{{ object.rules.count }}</td>
            </tr>
          </table>
        </div>
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

Here we've created a Boostrap 5 row and two column elements. In the first column, we have a simple card to display the access list's name and default action, as well as the number of rules assigned to it. And below it, you'll see an `include` tag which pulls in an additional template to render any custom fields associated with the model. In the second column, we've included two more templates to render tags and comments.

:green_circle: **Tip:** If you're not sure how best to construct the page's layout, there are plenty of examples to reference within NetBox's core templates.

Let's take a look at our new template! Navigate to the list view again (at <http://localhost:8000/plugins/access-lists/access-lists/>), and follow the link through to a particular access list. You should see something like the image below.

:blue_square: **Note:** If NetBox complains that the template still does not exist, you may need to manually restart the development server (`manage.py runserver`).

![Access list view](/images/step06-accesslist1.png)

This is nice, but it would be handy to include the access list's assigned rules on the page as well.

### Add a Rules Table

To include the access list rules, we'll need to provide additional _context data_ under the view. Open `views.py` and find the `AccessListView` class. (It should be the first class defined.) Add a `get_extra_context()` method to this class per the code below.

```python
class AccessListView(generic.ObjectView):
    queryset = models.AccessList.objects.all()

    def get_extra_context(self, request, instance):
        table = tables.AccessListRuleTable(instance.rules.all())
        table.configure(request)

        return {
            'rules_table': table,
        }
```

This method does three things:

1. Instantiate `AccessListRuleTable` with a queryset matching all rules assigned to this access list
2. Configure the table instance according to the current request (to honor user preferences)
3. Return a dictionary of context data referencing the table instance

This makes the table available to our template as the `rules_table` context variable. Let's add it to our template.

First, we need to import the `render_table` tag from the `django-tables2` library, so that we can render the table as HTML. Add this at the top of the template, immediately below the `{% extends 'generic/object.html' %}` line:

```
{% load render_table from django_tables2 %}
```

Then, immediately above the `{% endblock content %}` line at the end of the file, insert the following template code:

```
  <div class="row">
    <div class="col col-md-12">
      <div class="card">
        <h5 class="card-header">Rules</h5>
        <div class="card-body table-responsive">
          {% render_table rules_table %}
        </div>
      </div>
    </div>
  </div>
```

After refreshing the access list view in the browser, you should now see the rules table at the bottom of the page.

![Access list view with rules table](/images/step06-accesslist2.png)

## Create the AccessListRule Template

Speaking of rules, let's not forget about our `AccessListRule` model: It needs a template too. Create `accesslistrule.html` alongside our first template:

```bash
$ edit templates/netbox_access_lists/accesslistrule.html
```

And copy the content below:

```
{% extends 'generic/object.html' %}

{% block content %}
  <div class="row mb-3">
    <div class="col col-md-6">
      <div class="card">
        <h5 class="card-header">Access List Rule</h5>
        <div class="card-body">
          <table class="table table-hover attr-table">
            <tr>
              <th scope="row">Access List</th>
              <td>
                <a href="{{ object.access_list.get_absolute_url }}">{{ object.access_list }}</a>
              </td>
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
      </div>
      {% include 'inc/panels/custom_fields.html' %}
      {% include 'inc/panels/tags.html' %}
    </div>
    <div class="col col-md-6">
      <div class="card">
        <h5 class="card-header">Details</h5>
        <div class="card-body">
          <table class="table table-hover attr-table">
            <tr>
              <th scope="row">Protocol</th>
              <td>{{ object.get_protocol_display }}</td>
            </tr>
            <tr>
              <th scope="row">Source Prefix</th>
              <td>
                {% if object.source_prefix %}
                  <a href="{{ object.source_prefix.get_absolute_url }}">{{ object.source_prefix }}</a>
                {% else %}
                  {{ ''|placeholder }}
                {% endif %}
              </td>
            </tr>
            <tr>
              <th scope="row">Source Ports</th>
              <td>{{ object.source_ports|join:", "|placeholder }}</td>
            </tr>
            <tr>
              <th scope="row">Destination Prefix</th>
              <td>
                {% if object.destination_prefix %}
                  <a href="{{ object.destination_prefix.get_absolute_url }}">{{ object.destination_prefix }}</a>
                {% else %}
                  {{ ''|placeholder }}
                {% endif %}
              </td>
            </tr>
            <tr>
              <th scope="row">Destination Ports</th>
              <td>{{ object.destination_ports|join:", "|placeholder }}</td>
            </tr>
            <tr>
              <th scope="row">Action</th>
              <td>{{ object.get_action_display }}</td>
            </tr>
          </table>
        </div>
      </div>
    </div>
  </div>
{% endblock content %}
```

You'll probably be able to tell at this point what most of the above template code does, but here are a few details worth mentioning:

* The URL for the rule's parent access list is retrieved by calling `object.access_list.get_absolute_url()` (the method we added in step five), _without_ the parentheses (a distinction of DTL). This method is used for related prefixes as well.
* NetBox's `placeholder` filter is applied to the rule's description. (This renders a &mdash; for empty fields.)
* The `protocol` and `action` attributes are rendered by calling e.g. `object.get_protocol_display()` (again without the parentheses). This is a [Django convention](https://docs.djangoproject.com/en/stable/ref/models/instances/#extra-instance-methods) for static choice fields to return the human-friendly label rather than the raw value.

![Access list rule view](/images/step06-accesslistrule.png)

Feel free to experiment with different layouts and content before proceeding with the next step.

<div align="center">

:arrow_left: [Step 5: Views](/tutorial/step05-views.md) | [Step 7: Navigation](/tutorial/step07-navigation.md) :arrow_right:

</div>


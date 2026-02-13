# Step 5: Views

Views are responsible for the business logic in your application.
In practice, that usually means:

* receiving an incoming request
* fetching or updating data as needed
* returning a response to the client

Each view is typically associated with a URL, and it can handle one or more HTTP request methods (for example `GET` and `POST`).

Django includes a set of [generic view classes](https://docs.djangoproject.com/en/stable/topics/class-based-views/generic-display/) that take care of a lot of common request handling.
NetBox builds on that and provides its own view classes for common object workflows like viewing, listing, editing, and deleting.
These also integrate NetBox features such as custom fields and change logging.

In this step, we will create a set of views for each of our plugin models using the NetBox provided `register_model_view()` decorator.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step04-forms` (in case you've cloned the repository `netbox-plugin-demo`).

## Create the Views

Begin by creating `views.py` in the `netbox_access_lists/` directory of your plugin project root.

```bash
cd netbox_access_lists/
touch views.py
```

Open `views.py` and add the imports below.

We need:

* [NetBox's generic views](https://netboxlabs.com/docs/netbox/en/stable/plugins/development/views/#view-classes) base classes
* our plugin modules for forms, models, and tables
* Django's `Count` so we can annotate rule counts for the AccessList table

```python
from django.db.models import Count
from netbox.views import generic
from utilities.views import register_model_view

from . import forms, models, tables
```

:green_circle: **Tip:** We are importing the full `forms`, `models`, and `tables` modules. If you prefer importing specific classes instead, that is totally fine. Update the view definitions accordingly.

For each model, we will create four views:

* **Detail view**: display a single object
* **List view**: display a table of all existing instances of a particular model
* **Edit view**: handle adding and modifying objects
* **Delete view**: handle the deletion of an object

For reference, your plugin project should now include `views.py`:

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
│   └── views.py
├── pyproject.toml
└── README.md
```

## About `register_model_view`

NetBox provides the `@register_model_view()` decorator to register views with the model view registry.
NetBox uses this registry to determine which views exist for a model and which actions should appear in the UI.

This decorator is optional. If you prefer, you can register your views manually.
Using the decorator tends to keep things consistent and reduces boilerplate.

A few decorator arguments you will see in this step:

* `name` is used as part of the view name for URL reversing
* `path` controls the URL path fragment for the view (optional)
* `detail` should be `True` for views tied to a specific object and `False` for views attached to the list path

More details are available in the NetBox documentation: [view URL registration](https://netboxlabs.com/docs/netbox/plugins/development/views/#url-registration-1).

## AccessList Views

The general pattern we will follow here is to subclass a generic view class provided by NetBox and define the necessary attributes.
We will not need to write much custom logic because the NetBox provided views handle most of the request flow for us.

### Detail view

The detail view shows a single object.
We use `generic.ObjectView` and provide a queryset.

```python
@register_model_view(models.AccessList)
class AccessListView(generic.ObjectView):
    queryset = models.AccessList.objects.all()
```

:green_circle: **Tip:** NetBox views expect a queryset rather than just a model. That makes it easy to add optimizations later, like `prefetch_related()` or annotations.

### List view

The list view shows a table of objects.
We need both `queryset` and `table`.

In Step 3 we added a `rule_count` column to the AccessList table.
That column expects a count of rules for each access list in the queryset, named `rule_count`.

We can use Django's [`Count()`](https://docs.djangoproject.com/en/stable/ref/models/querysets/#aggregation-functions) function to annotate the number of associated rules.

```python
@register_model_view(models.AccessList, name='list', path='', detail=False)
class AccessListListView(generic.ObjectListView):
    queryset = models.AccessList.objects.annotate(
        rule_count=Count('rules'),
    )
    table = tables.AccessListTable
```

:green_circle: **Tip:** The names `AccessListView` and `AccessListListView` are easy to mix up at a glance. `AccessListView` is the detail view (one object). `AccessListListView` is the list view (many objects).

### Edit and delete views

The edit view handles both add and edit actions.
We register it twice:

* `add` registers the view at the collection level (not tied to a specific object)
* `edit` registers the view for a specific object

```python
@register_model_view(models.AccessList, name='add', detail=False)
@register_model_view(models.AccessList, name='edit')
class AccessListEditView(generic.ObjectEditView):
    queryset = models.AccessList.objects.all()
    form = forms.AccessListForm
```

The delete view is straightforward:

```python
@register_model_view(models.AccessList, name='delete')
class AccessListDeleteView(generic.ObjectDeleteView):
    queryset = models.AccessList.objects.all()
```

## AccessListRule Views

The `AccessListRule` views follow the same pattern.
Adding a short comment block can make the file easier to scan later.

```python
#
# AccessListRule views
#

@register_model_view(models.AccessListRule)
class AccessListRuleView(generic.ObjectView):
    queryset = models.AccessListRule.objects.all()


@register_model_view(models.AccessListRule, name='list', path='', detail=False)
class AccessListRuleListView(generic.ObjectListView):
    queryset = models.AccessListRule.objects.all()
    table = tables.AccessListRuleTable


@register_model_view(models.AccessListRule, name='add', detail=False)
@register_model_view(models.AccessListRule, name='edit')
class AccessListRuleEditView(generic.ObjectEditView):
    queryset = models.AccessListRule.objects.all()
    form = forms.AccessListRuleForm


@register_model_view(models.AccessListRule, name='delete')
class AccessListRuleDeleteView(generic.ObjectDeleteView):
    queryset = models.AccessListRule.objects.all()
```

With our views in place, the next step is to make them reachable by associating them with URLs.

<div align="center">

:arrow_left: [Step 4: Forms](/tutorial/step04-forms.md) | [Step 6: URLs](/tutorial/step06-urls.md) :arrow_right:

</div>

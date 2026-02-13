# Step 9: Filter Sets

Filters let users request only a specific subset of objects matching a query, for example, filtering the Sites list by status or region.

NetBox uses the [django filters](https://django-filter.readthedocs.io/en/stable/) library to build and apply filter sets for models.
In this step we will create a filter set for our plugin models, so the list view and REST API can apply the same kind of filtering.
For brevity, we will only create a filter set for `AccessListRule`, but the same approach applies to `AccessList` too.

In this step we will build:

* a **filter set** for `AccessListRule` (the query logic)
* a **filter form** so the UI can render a nice filter panel

:blue_square: **Note:** If you skipped the previous step, and you cloned the demo repository, run `git checkout step08-navigation`.

## Create a filter set

Begin by creating `filtersets.py` in the `netbox_access_lists/` directory of your plugin project root.

```bash
cd netbox_access_lists/
touch filtersets.py
```

### Start with a basic FilterSet

Open `filtersets.py` and add the imports below. We will import:

* `NetBoxModelFilterSet` as our base class
* `AccessListRule` as the model we want to filter
* `django_filters` because we will define some custom filters
* `Prefix` because our model references it
* `NumericArrayFilter` because our ports are stored as integer arrays

```python
import django_filters

from ipam.models import Prefix
from netbox.filtersets import NetBoxModelFilterSet
from utilities.filters import NumericArrayFilter

from .models import AccessListRule
```

Next, create a class named `AccessListRuleFilterSet` that subclasses `NetBoxModelFilterSet`.

Inside it, define a `Meta` class and set:

* `model` to the model we are filtering
* `fields` to the list of model fields we want NetBox and django filters to expose it automatically

This should feel familiar if you have already built a model form.
The idea is very similar.

```python
class AccessListRuleFilterSet(NetBoxModelFilterSet):

    class Meta:
        model = AccessListRule
        fields = ('id', 'access_list', 'index', 'protocol', 'action')
```

`NetBoxModelFilterSet` handles some important functions for us, including support for filtering by custom field values and tags.

:warning: **Warning:** NetBox relies on a specific naming convention for filter sets.

The module must be named `filtersets`, and the FilterSet class name must match the model name with `FilterSet` appended.
For example, the model `netbox_access_lists.models.AccessList` must have a filter set class accessible as `netbox_access_lists.filtersets.AccessListFilterSet`.

Other naming schemes may appear to work at first, but certain NetBox features can break, including GraphQL filtering and selectors for dynamic model fields.

### Add a simple search implementation

`NetBoxModelFilterSet` also provides a general purpose `q` filter.
When a user enters a value in the search box, NetBox calls the filter set `search()` method.
By default, `search()` does nothing, so we will override it.

Add this method under the `Meta` class:

```python
    def search(self, queryset, name, value):
        return queryset.filter(description__icontains=value)
```

This returns all rules whose description contains the search string.
You can expand this later to match other fields as well.

### Add Prefix Filters

Our model has two foreign key fields:

* `source_prefix`
* `destination_prefix`

Both point to NetBox's `Prefix` model. We want to let users filter rules by prefix in two ways:

* by the prefix value itself (example `192.0.2.0/24`)
* by the prefix ID (useful for API users and for dynamic form widgets)

To do this, we will use `django_filters.ModelMultipleChoiceFilter`.
This filter type allows users to pass one or more values.

Add these filters to the `AccessListRuleFilterSet` class, above the `Meta` class and update the `fields` list:

```python
    source_prefix = django_filters.ModelMultipleChoiceFilter(
        field_name='source_prefix__prefix',
        queryset=Prefix.objects.all(),
        to_field_name='prefix',
        label='Source Prefix (value)',
    )
    source_prefix_id = django_filters.ModelMultipleChoiceFilter(
        field_name='source_prefix',
        queryset=Prefix.objects.all(),
        to_field_name='id',
        label='Source Prefix (ID)',
    )
    destination_prefix = django_filters.ModelMultipleChoiceFilter(
        field_name='destination_prefix__prefix',
        queryset=Prefix.objects.all(),
        to_field_name='prefix',
        label='Destination Prefix (value)',
    )
    destination_prefix_id = django_filters.ModelMultipleChoiceFilter(
        field_name='destination_prefix',
        queryset=Prefix.objects.all(),
        to_field_name='id',
        label='Destination Prefix (ID)',
    )

    class Meta:
        model = AccessListRule
        fields = ('id', 'access_list', 'index', 'protocol', 'action')
```

A quick explanation of the key arguments:

* `field_name` tells django filters which field to filter on
  * it can traverse relationships using Django double underscore syntax
  * example: `source_prefix__prefix` means follow `source_prefix` to the related `Prefix` object, then use its `prefix` field
* `queryset` is used to look up valid related objects
* `to_field_name` tells the filter which field on the related model should match the user supplied values

:blue_square: **Note:** We are defining `source_prefix_id` and `destination_prefix_id` as custom filters. Because they are explicitly declared on the class, they do not need to be listed in `Meta.fields`.

:blue_square: **Note:** When filtering on a field of a related model, you will usually use double underscore traversal in `field_name`. The value of `to_field_name` should match the field you want to accept as input, like `id` or `prefix`.

### Filter by source and destination ports

The model fields `source_ports` and `destination_ports` are stored as a list of integers, so we will use `NumericArrayFilter` to filter by values contained in those lists.

We want a simple behavior: return rules where the list contains a specific port number. That is why we use `lookup_expr="contains"`.

Add these filters to the `AccessListRuleFilterSet` class:

```python
    source_port = NumericArrayFilter(
        field_name='source_ports',
        lookup_expr='contains',
        label='Source Port',
    )
    destination_port = NumericArrayFilter(
        field_name='destination_ports',
        lookup_expr='contains',
        label='Destination Port',
    )
```

:blue_square: **Note:** Just like the prefix ID filters, `source_port` and `destination_port` are custom filters. Because they are explicitly declared on the class, they do not need to be listed in `Meta.fields`.

### Full `filtersets.py` so far

At this point, your `filtersets.py` should look like this:

```python
import django_filters

from ipam.models import Prefix
from netbox.filtersets import NetBoxModelFilterSet
from utilities.filters import NumericArrayFilter

from .models import AccessListRule


class AccessListRuleFilterSet(NetBoxModelFilterSet):
    source_prefix = django_filters.ModelMultipleChoiceFilter(
        field_name='source_prefix__prefix',
        queryset=Prefix.objects.all(),
        to_field_name='prefix',
        label='Source Prefix (value)',
    )
    source_prefix_id = django_filters.ModelMultipleChoiceFilter(
        field_name='source_prefix',
        queryset=Prefix.objects.all(),
        to_field_name='id',
        label='Source Prefix (ID)',
    )
    destination_prefix = django_filters.ModelMultipleChoiceFilter(
        field_name='destination_prefix__prefix',
        queryset=Prefix.objects.all(),
        to_field_name='prefix',
        label='Destination Prefix (value)',
    )
    destination_prefix_id = django_filters.ModelMultipleChoiceFilter(
        field_name='destination_prefix',
        queryset=Prefix.objects.all(),
        to_field_name='id',
        label='Destination Prefix (ID)',
    )

    source_port = NumericArrayFilter(
        field_name='source_ports',
        lookup_expr='contains',
        label='Source Port',
    )
    destination_port = NumericArrayFilter(
        field_name='destination_ports',
        lookup_expr='contains',
        label='Destination Port',
    )

    class Meta:
        model = AccessListRule
        fields = ('id', 'access_list', 'index', 'protocol', 'action')

    def search(self, queryset, name, value):
        return queryset.filter(description__icontains=value)
```

For reference, your plugin project should now include `filtersets.py`:

```text
.
├── netbox_access_lists
│   ├── choices.py
│   ├── filtersets.py
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

### Enable advanced filtering

NetBox provides a mechanism for plugins to register filter sets for use in the advanced filtering UI (sometimes referenced as filter modifiers).
This can be enabled by adding a decorator to the filter set class.

First, import the `register_filterset` decorator from `utilities.filtersets` at the top of `filtersets.py`:

```python
from utilities.filtersets import register_filterset
```

Then add the decorator to each FilterSet class, like `AccessListRuleFilterSet`:

```python
@register_filterset
class AccessListRuleFilterSet(NetBoxModelFilterSet):
    # ...
```

:blue_square: **Note:** This is a NetBox v4.5 feature. If you are reading this tutorial while using an older NetBox version, this decorator will not be available.

## Create a Filter Form

The filter set defines how filtering works, but it does not automatically create the UI fields for the NetBox filter panel.
For that we need a filter form class.

We will add this to `forms.py`.

### Update imports in `forms.py`

Open `forms.py` and make the following updates:

* import Django `forms`
* import [`NetBoxModelFilterSetForm`](https://netboxlabs.com/docs/netbox/en/stable/plugins/development/forms/#netboxmodelfiltersetform)
* import the ChoiceSets for protocol and action
* import `Prefix` and `AccessList` because our filter fields need querysets
* import a few NetBox utility fields for dynamic selection and tags

Add or extend your imports so they include:

```python
from django import forms

from ipam.models import Prefix
from netbox.forms import NetBoxModelFilterSetForm, NetBoxModelForm
from utilities.forms.fields import (
    CommentField,
    DynamicModelChoiceField,
    DynamicModelMultipleChoiceField,
    TagFilterField,
)

from .choices import ActionChoices, ProtocolChoices
from .models import AccessList, AccessListRule
```

:blue_square: **Note:** You might already have some of these imports from earlier steps. If so, you only need to add the missing ones.

### Create `AccessListRuleFilterForm`

Create a form class named `AccessListRuleFilterForm` subclassing `NetBoxModelFilterSetForm`.
Unlike `NetBoxModelForm`, a filter form does not need a `Meta` class. It just needs a `model` attribute.

```python
class AccessListRuleFilterForm(NetBoxModelFilterSetForm):
    model = AccessListRule
```

Next, define a form field for each filter you want to appear in the UI.
Every field should use `required=False`, because filters are optional.

### Add basic fields

Start with the `access_list` filter. This references a related object, so we will use `ModelMultipleChoiceField` to allow filtering by multiple objects:

```python
    access_list = forms.ModelMultipleChoiceField(
        queryset=AccessList.objects.all(),
        required=False,
    )
```

:blue_square: **Note:** We are using Django `ModelMultipleChoiceField` for this field instead of NetBox `DynamicModelMultipleChoiceField` because the dynamic field requires a functional REST API endpoint for the model. Once we implement the plugin REST API in Step 10, you can revisit this form and switch `access_list` to a dynamic field.

Next, add a field for the `index` filter:

```python
    index = forms.IntegerField(
        required=False,
    )
```

For the `protocol` and `action` fields, we want choice based filters.
Use `MultipleChoiceField` to allow selecting one or more choices:

```python
    protocol = forms.MultipleChoiceField(
        choices=ProtocolChoices,
        required=False,
    )
    action = forms.MultipleChoiceField(
        choices=ActionChoices,
        required=False,
    )
```

Finally, add fields for the ports:

```python
    source_port = forms.IntegerField(
        label='Source Port',
        required=False,
    )
    destination_port = forms.IntegerField(
        label='Destination Port',
        required=False,
    )
```

### Add dynamic prefix fields

Even though we cannot use `DynamicModelMultipleChoiceField` for `access_list` yet, we can use it for prefixes because NetBox already has a REST API endpoint for `Prefix`.

The dynamic field returns prefix IDs, so we will use the ID based filters `source_prefix_id` and `destination_prefix_id` in the form.

```python
    source_prefix_id = DynamicModelMultipleChoiceField(
        queryset=Prefix.objects.all(),
        required=False,
        label='Source Prefix',
    )
    destination_prefix_id = DynamicModelMultipleChoiceField(
        queryset=Prefix.objects.all(),
        required=False,
        label='Destination Prefix',
    )
```

### Add tag filtering

Filtering by tags is a special case. This requires a custom form field:

```python
    tag = TagFilterField(model)
```

### Group filter fields

NetBox provides a mechanism for grouping filter fields into logical sections.
This is optional, but it can make the filter form easier to scan.

Grouping is achieved by adding a `fieldsets` attribute to the form class.
This attribute is a tuple of `FieldSet` objects, where each `FieldSet` contains an optional section name and a list of field names.
The section name is displayed as a header in the filter form.

First, import `FieldSet`:

```python
from utilities.forms.rendering import FieldSet
```

Then add `fieldsets` to the form class:

```python
class AccessListRuleFilterForm(NetBoxModelFilterSetForm):
    model = AccessListRule
    fieldsets = (
        FieldSet(
            'q',
            'filter_id',
            'tag',
        ),
        FieldSet(
            'access_list',
            'index',
            'protocol',
            'action',
            name='Attributes',
        ),
        FieldSet(
            'source_prefix_id',
            'source_port',
            name='Source',
        ),
        FieldSet(
            'destination_prefix_id',
            'destination_port',
            name='Destination',
        ),
    )
    # the fields continue below
```

You probably noticed that we also added `filter_id`.
This enables the Saved Filters feature in the UI. The `q` and `filter_id` fields are provided by the base form.

## Update the View

Now we need to tell the list view to use our filter set and filter form.

Open `views.py` and extend the existing imports to include the `filtersets` module:

```bash
edit views.py
```

```python
from . import filtersets, forms, models, tables
```

Then add the `filterset` and `filterset_form` attributes to `AccessListRuleListView`:

```python
@register_model_view(models.AccessListRule, name='list', path='', detail=False)
class AccessListRuleListView(generic.ObjectListView):
    queryset = models.AccessListRule.objects.all()
    table = tables.AccessListRuleTable
    filterset = filtersets.AccessListRuleFilterSet
    filterset_form = forms.AccessListRuleFilterForm
```

After ensuring the development server has restarted, navigate to the rules list view in the browser.
You should now see a Filters tab next to the Results tab.
Under it, you will find the fields you created on `AccessListRuleFilterForm`, as well as the search field.

![Access list rules filter form](/images/step09-filter-form.png)

If you have not already, create a few more access lists and rules, and experiment with the filters.
Consider how you might filter by additional fields or add more complex logic to the filter set.

:green_circle: **Tip:** You may notice that we did not add a form field for the model `id` filter. This is because it is unlikely to be useful for a human using the UI. However, we still want to support filtering objects by their primary keys because it is very helpful for consumers of the NetBox REST API, which we will cover next.

<div align="center">

:arrow_left: [Step 8: Navigation](/tutorial/step08-navigation.md) | [Step 10: REST API](/tutorial/step10-rest-api.md) :arrow_right:

</div>

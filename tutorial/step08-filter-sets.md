# Step 8: Filter Sets

Filters enable users to request only a specific subset of objects matching a query; when filtering the sites list by status or region, for instance. NetBox employs the [`django-filters`](https://django-filter.readthedocs.io/en/stable/) library to build and apply filter sets for models. We can create filter sets to enable this same functionality for our plugin as well.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step07-navigation`.

## Create a Filter Set

Begin by creating `filtersets.py` in the `netbox_access_lists/` directory.

```bash
$ cd netbox_access_lists/
$ edit filtersets.py
```

At the top of this file, we'll import  NetBox's `NetBoxModelFilterSet` class, which will serve as the base class for our filter set, as well as our `AccessListRule` model. (In the interest of brevity, we're only going to create a filter set for one model, but it should be clear how to replicate this approach for the `AccessList` model as well.)

```python
from netbox.filtersets import NetBoxModelFilterSet
from .models import AccessListRule
```

Next, create a class named `AccessListRuleFilterSet` subclassing `NetBoxModelFilterSet`. Within this class, create a child `Meta` class and define the filter set's `model` and `fields` attributes. (You may notice this looks familiar; it is very similar to the process for building a model form.) The `fields` parameter should list all the model fields against which we might want to filter.

```python
class AccessListRuleFilterSet(NetBoxModelFilterSet):

    class Meta:
        model = AccessListRule
        fields = ('id', 'access_list', 'index', 'protocol', 'action')
```

`NetBoxModelFilterSet` handles some important functions for us, including support for filtering by custom field values and tags. It also creates a general-purpose `q` filter which invokes the `search()` method. (By default, this does nothing.) We can override this method to define our general-purpose search logic. Let's add a `search` method after the `Meta` child class to override the default behavior.

```python
    def search(self, queryset, name, value):
        return queryset.filter(description__icontains=value)
```

This will return all rules whose description contains the queried string. Of course, you're free to extend this to match other fields as well, but for our purposes this should be sufficient.

## Create a Filter Form

The filter set handles the "behind the scenes" process of filtering queries, but we also need to create a form class to render the filter fields in the UI. We'll add this to `forms.py`. First, import Django's `forms` module (which will provide the field classes we need) and append `NetBoxModelFilterSetForm` to the existing import statement for `netbox.forms`:

```python
from django import forms
# ...
from netbox.forms import NetBoxModelForm, NetBoxModelFilterSetForm
```

Then create a form class named `AccessListRuleFilterForm` subclassing `NetBoxModelFilterSetForm` and declare an attribute named `model` referencing `AccessListRule` (which has already been imported for one of the existing forms).

```python
class AccessListRuleFilterForm(NetBoxModelFilterSetForm):
    model = AccessListRule
```

:blue_square: **Note:** Note that the `model` attribute is declared directly under the class: We don't need a `Meta` child class.

Next, we need to define a form field for each filter that we want to appear in the UI. Let's start with the `access_list` filter: This references a related object, so we'll want to use `ModelMultipleChoiceField` (to allow users to filter by multiple objects). Add the form field with the same name as its peer filter, specifying the queryset to use when fetching related objects.

```python
    access_list = forms.ModelMultipleChoiceField(
        queryset=AccessList.objects.all(),
        required=False
    )
```

Notice that we've also set `required=False`: This should be the case for _all_ fields in a filter form, because filters are never mandatory.

:blue_square: **Note:** We're using Django's `ModelMultipleChoiceField` class for this field instead of NetBox's `DynamicModelChoiceField` because the latter requires a functional REST API endpoint for the model. Once we implement a REST API in step nine, you're free to revisit this form and change `access_list` to a `DynamicModelChoiceField`.

Next we'll add a field for the `position` filter: This is an integer field, so `IntegerField` should work nicely:

```python
    index = forms.IntegerField(
        required=False
    )
```

Finally, we'll add fields for the `protocol` and `action` choice-based filters. `MultipleChoiceField` should be used to allow users to select one or more choices. We must pass the set of valid choices when declaring these fields, so first extend the relevant import statement at the top of `forms.py`:

```python
from .models import AccessList, AccessListRule, ActionChoices, ProtocolChoices
```

Then add the form fields to `AccessListRuleFilterForm`:

```python
    protocol = forms.MultipleChoiceField(
        choices=ProtocolChoices,
        required=False
    )
    action = forms.MultipleChoiceField(
        choices=ActionChoices,
        required=False
    )
```

## Update the View

The last step before we can use our new filter set and form is to enable them under the model's list view. Open `views.py` and extend the last import statement to include the `filtersets` module:

```python
from . import filtersets, forms, models, tables
```

Then, add the `filterset` and `filterset_form` attributes to `AccessListRuleListView`:

```python
class AccessListRuleListView(generic.ObjectListView):
    queryset = models.AccessListRule.objects.all()
    table = tables.AccessListRuleTable
    filterset = filtersets.AccessListRuleFilterSet
    filterset_form = forms.AccessListRuleFilterForm
```

After ensuring the development server has restarted, navigate to the rules list view in the browser. You should now see a "Filters" tab next to the "Results" tab. Under it we'll find the four fields we created on `AccessListRuleFilterForm`, as well as the built-in "search" field.

![Access list rules filter form](/images/step08-filter-form.png)

If you haven't already, create a few more access lists and rules, and experiment with the filters. Consider how you might filter by additional fields, or add more complex logic to the filter set.

:green_circle: **Tip:** You may notice that we did not add a form field for the model's `id` filter: This is because it is unlikely to be useful for a human utilizing the UI. However, we still want to support filtering object by their primary keys, because it _is_ very helpful for consumers of NetBox's REST API, which we'll cover next.

<div align="center">

:arrow_left: [Step 7: Navigation](/tutorial/step07-navigation.md) | [Step 9: REST API](/tutorial/step09-rest-api.md) :arrow_right:

</div>


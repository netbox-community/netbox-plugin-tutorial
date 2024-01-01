# Step 5: Views

Views are responsible for the business logic of your application. Generally, this means processing incoming requests, performing some action(s), and returning a response to the client. Each view typically has a URL associated with it, and can handle one or more types of HTTP requests (i.e. `GET` and/or `POST` requests).

Django provides a set of [generic view classes](https://docs.djangoproject.com/en/4.0/topics/class-based-views/generic-display/) which handle much of the boilerplate code needed to process requests. NetBox likewise provides a set of view classes to simplify the creation of views for creating, editing, deleting, and viewing objects. They also introduce support for NetBox-specific features such as custom fields and change logging.

In this step, we'll create a set of views for each of our plugin's models.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step04-forms`.

## Create the Views

Begin by creating `views.py` in the `netbox_access_lists/` directory.

```bash
$ cd netbox_access_lists/
$ edit views.py
```

We'll need to import our plugin's `models`, `tables`, and `forms` modules: This is where everything we've built so far really comes together! We also need to import NetBox's generic views module, as it provides the base classes for our views.

```python
from netbox.views import generic
from . import forms, models, tables
```

:green_circle: **Tip:** You'll notice that we're importing the entire model, form, and tables modules here. If you would prefer to import each of the relevant classes directly, you're certainly welcome to do so; just remember to change the class definitions below accordingly.

For each model, we need to create four views:

* **Detail view** - Display a single object
* **List view** - Displays a table of all existing instances of a particular model
* **Edit view** - Handles adding and modifying objects
* **Delete view** - Handles the deletion of an object

### AccessList Views

The general pattern we'll follow here is to subclass a generic view class provided by NetBox, and define the necessary attributes. We won't need to write any substantial code because the views NetBox provides takes care of the request logic for us.

Let's start with a detail view. We subclass `generic.ObjectView` and define the queryset of objects we want to display.

```python
class AccessListView(generic.ObjectView):
    queryset = models.AccessList.objects.all()
```

:green_circle: **Tip:** The views require us to define a queryset rather than just a model, because it's sometimes necessary to modify the queryset, e.g. to prefetch related objects or limit by a particular attribute.

Next, we'll add a list view. For this view, we need to define both `queryset` and `table`.

```python
class AccessListListView(generic.ObjectListView):
    queryset = models.AccessList.objects.all()
    table = tables.AccessListTable
```

:green_circle: **Tip:** It occurs to the author that having chosen a model name that ends with "List" might be a bit confusing here. Just remember that `AccessListView` is the _detail_ (single object) view, and `AccessListListView` is the _list_ (multiple objects) view.

Before we move on to the next view, do you remember the extra column we added to `AccessListTable` in step three? That column expects to find a count of rules assigned for each access list in the queryset, named `rule_count`. Let's add this to our queryset now. We can employ Django's `Count()` function to extend the SQL query and annotate the count of associated rules. (Don't forget to add the import statement up top.)

```python
from django.db.models import Count
# ...
class AccessListListView(generic.ObjectListView):
    queryset = models.AccessList.objects.annotate(
        rule_count=Count('rules')
    )
    table = tables.AccessListTable
```

We'll finish up with the edit and delete views for `AccessList`. Note that for the edit view, we also need to define `form` as the form class we created in step four.

```python
class AccessListEditView(generic.ObjectEditView):
    queryset = models.AccessList.objects.all()
    form = forms.AccessListForm

class AccessListDeleteView(generic.ObjectDeleteView):
    queryset = models.AccessList.objects.all()
```

That's it for the first model! We'll create another four views for `AccessListRule` as well.

### AccessListRule Views

The rest of our views follow the same pattern as the first four.

```python
class AccessListRuleView(generic.ObjectView):
    queryset = models.AccessListRule.objects.all()


class AccessListRuleListView(generic.ObjectListView):
    queryset = models.AccessListRule.objects.all()
    table = tables.AccessListRuleTable


class AccessListRuleEditView(generic.ObjectEditView):
    queryset = models.AccessListRule.objects.all()
    form = forms.AccessListRuleForm


class AccessListRuleDeleteView(generic.ObjectDeleteView):
    queryset = models.AccessListRule.objects.all()
```

With our views in place, we now need to make them accessible by associating each with a URL.

## Map Views to URLs

In the `netbox_access_lists/` directory, create `urls.py`. This will define our view URLs.

```bash
$ edit urls.py
```

URL mapping for NetBox plugins is pretty much identical to regular Django apps: We'll define `urlpatterns` as an iterable of `path()` calls, mapping URL fragments to view classes.

First we'll need to import Django's `path` function from its `urls` module, as well as our plugin's `models` and `views` modules.

```python
from django.urls import path
from . import models, views
```

We have four views per model, but we actually need to define five paths for each. This is because the add and edit operations are handled by the same view, but accessed via different URLs. Along with the URL and view for each path, we'll also specify a `name`; this allows us to easily reference a URL in code.

```python
urlpatterns = (
    path('access-lists/', views.AccessListListView.as_view(), name='accesslist_list'),
    path('access-lists/add/', views.AccessListEditView.as_view(), name='accesslist_add'),
    path('access-lists/<int:pk>/', views.AccessListView.as_view(), name='accesslist'),
    path('access-lists/<int:pk>/edit/', views.AccessListEditView.as_view(), name='accesslist_edit'),
    path('access-lists/<int:pk>/delete/', views.AccessListDeleteView.as_view(), name='accesslist_delete'),
)
```

We've chosen `access-lists` as the base URL for our `AccessList` model, but you are free to choose something different. However, it is recommended to retain the naming scheme shown, as several NetBox features rely on it. Also note that each of the views must be invoked by its `as_view()` method when passed to `path()`.

:green_circle: **Tip:** The `<int:pk>` string you see in some of the URLs is a [path converter](https://docs.djangoproject.com/en/stable/topics/http/urls/#path-converters). Specifically, this is an integer (`int`) variable named `pk`. This value is extracted from the request URL and passed to the view when the request is processed, so that the specified object can be located in the database.

Let's add the rest of the paths now. You may find it helpful to separate the paths by model to make the file more readable.

```python
urlpatterns = (

    # Access lists
    path('access-lists/', views.AccessListListView.as_view(), name='accesslist_list'),
    path('access-lists/add/', views.AccessListEditView.as_view(), name='accesslist_add'),
    path('access-lists/<int:pk>/', views.AccessListView.as_view(), name='accesslist'),
    path('access-lists/<int:pk>/edit/', views.AccessListEditView.as_view(), name='accesslist_edit'),
    path('access-lists/<int:pk>/delete/', views.AccessListDeleteView.as_view(), name='accesslist_delete'),

    # Access list rules
    path('rules/', views.AccessListRuleListView.as_view(), name='accesslistrule_list'),
    path('rules/add/', views.AccessListRuleEditView.as_view(), name='accesslistrule_add'),
    path('rules/<int:pk>/', views.AccessListRuleView.as_view(), name='accesslistrule'),
    path('rules/<int:pk>/edit/', views.AccessListRuleEditView.as_view(), name='accesslistrule_edit'),
    path('rules/<int:pk>/delete/', views.AccessListRuleDeleteView.as_view(), name='accesslistrule_delete'),

)
```

### Adding Changelog Views

You may recall that one of the features provided by NetBox is automatic [change logging](https://netbox.readthedocs.io/en/stable/additional-features/change-logging/). You can see this in action when viewing a NetBox object and selecting its "Changelog" tab. Since our models inherit from `NetBoxModel`, they too can utilize this feature.

We'll add a dedicated changelog URL for each of our models. First, back at the top of `urls.py`, we need to import NetBox's `ObjectChangeLogView`:

```python
from netbox.views.generic import ObjectChangeLogView
```

Then, we'll add an extra path for each model inside `urlpatterns`:

```python
urlpatterns = (

    # Access lists
    # ...
    path('access-lists/<int:pk>/changelog/', ObjectChangeLogView.as_view(), name='accesslist_changelog', kwargs={
        'model': models.AccessList
    }),

    # Access list rules
    # ...
    path('rules/<int:pk>/changelog/', ObjectChangeLogView.as_view(), name='accesslistrule_changelog', kwargs={
        'model': models.AccessListRule
    }),

)
```

Notice that we're using `ObjectChangeLogView` directly here; we did not need to create model-specific subclasses for it. Additionally, we're passing a keyword argument `model` to the view: This specifies the model to be used (which is why we didn't need to subclass the view).

## Add Model URL Methods

Now that we have our URL paths in place, we can add a `get_absolute_url()` method to each of our models. The method is a [Django convention](https://docs.djangoproject.com/en/stable/ref/models/instances/#get-absolute-url); although not strictly required, it conveniently returns the absolute URL for any particular object. For example, calling `accesslist.get_absolute_url()` would return `/plugins/access-lists/access-lists/123/` (where 123 is the primary key of the object).

Back in `models.py`, import Django's `reverse` function from its `urls` module at the top of the file:

```python
from django.urls import reverse
```

Then, add the `get_absolute_url()` method to the `AccessList` class after its `__str__()` method:

```python
class AccessList(NetBoxModel):
    # ...
    def get_absolute_url(self):
        return reverse('plugins:netbox_access_lists:accesslist', args=[self.pk])
```

`reverse()` takes two arguments here: The view name, and a list of positional arguments. The view name is formed by concatenating three components:

* The string `'plugins'`
* The name of our plugin
* The name of the desired URL path (defined as `name='accesslist'` in `urls.py`)

The object's `pk` attribute is passed as well, and replaces the `<int:pk>` path converter in the URL.

We'll add a `get_absolute_url()` method for `AccessListRule` as well, adjusting the view name accordingly.

```python
class AccessListRule(NetBoxModel):
    # ...
    def get_absolute_url(self):
        return reverse('plugins:netbox_access_lists:accesslistrule', args=[self.pk])
```

## Test the Views

Now for the moment of truth: Has all our work thus far yielded functional UI views? Check that the development server is running, then open a browser and navigate to <http://localhost:8000/plugins/access-lists/access-lists/>. You should see the access list list view and (if you followed in step two) a single access list named MyACL1.

:blue_square: **Note:** This guide assumes that you're running the Django development server locally on port 8000. If your setup is different, you'll need to adjust the link above accordingly.

![Access lists list view](/images/step05-accesslist-list.png)

We see that our table has successfully render the `name`, `rule_count`, and `default_action` columns that we defined in step three, and the `rule_count` column shows two rules assigned as expected.

If we click the "Add" button at top right, we'll be taken to the access list creation form. (Creating a new access list won'r work yet, but the form should render as seen below.)

![Access list creation form](/images/step05-accesslist-form.png)

However, if you click a link to an access list in the table, you'll be met by a `TemplateDoesNotExist` exception. This means exactly what it says: We have not yet defined a template for this view. Don't worry, that's coming up next!

:blue_square: **Note:** You might notice that the "add" view for rules still doesn't work, raising a `NoReverseMatch` exception. This is because we haven't yet defined the REST API backends required to support the dynamic form fields. We'll take care of this when we build out the REST API functionality in step nine.

<div align="center">

:arrow_left: [Step 4: Forms](/tutorial/step04-forms.md) | [Step 6: Templates](/tutorial/step06-templates.md) :arrow_right:

</div>


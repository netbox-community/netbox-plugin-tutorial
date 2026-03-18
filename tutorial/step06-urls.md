# Step 6: URLs

The views we created in the previous step exist as Python classes, but NetBox still needs to know which URL paths should route to which views. In this step we will map our views to URLs in one of two ways:

* Use NetBox's `get_model_urls()` helper (recommended, works great with `register_model_view()`)
* Define every URL path manually (useful if you prefer explicit routing)

This guide assumes you are using the decorator-based approach, but the manual approach is included as an option.

:blue_square: **Note:** If you skipped the previous step, and you cloned the demo repository, run `git checkout step05-views` (in case you've cloned the repository `netbox-plugin-demo`).

## Map Views to URLs using the decorator

In the `netbox_access_lists/` directory of your plugin project root, create `urls.py`. This file defines the URL patterns for your plugin.

```bash
cd netbox_access_lists/
touch urls.py
```

Open `urls.py` and add the imports below.

We need:

* Django's `include()` and `path()`
* NetBox's `get_model_urls()` helper
* our `views` module (important, so the decorators execute and register views)

```python
from django.urls import include, path
from utilities.urls import get_model_urls

from . import views
```

:green_circle: **Tip:** `views` is not referenced directly in this file, but importing it ensures the `@register_model_view()` decorators run and populate the model view registry.

### Add AccessList URLs

`get_model_urls()` can generate all URL patterns for a model based on the views registered via `register_model_view()`. In practice, you will usually include it twice for each model:

* once for list level routes (where `detail=False`)
* once for object detail routes (where the path contains `<int:pk>`)

Add the following to `urls.py`:

```python
urlpatterns = (
    # Access lists
    path(
        'access-lists/',
        include(get_model_urls('netbox_access_lists', 'accesslist', detail=False)),
    ),
    path(
        'access-lists/<int:pk>/',
        include(get_model_urls('netbox_access_lists', 'accesslist')),
    ),
)
```

We chose `access lists` as the base URL path for the `AccessList` model. You can choose something else, but keeping a consistent scheme makes it easier to reason about your plugin and aligns with how NetBox structures many of its own URLs.

`get_model_urls()` takes these arguments:

* `app_label`: the label of the app containing the model (for example `netbox_access_lists`)
* `model_name`: the model name in lowercase (for example `accesslist`)
* `detail`: `True` if the routes apply to a specific object, `False` for list level routes

:green_circle: **Tip:** `<int:pk>` is a [path converter](https://docs.djangoproject.com/en/stable/topics/http/urls/#path-converters). It captures an integer named `pk` from the URL and passes it to the view so the correct object can be retrieved.

:blue_square: **Note:** This approach also registers additional model features when available (for example, changelog and journal entries), as long as the relevant views exist.

### Add AccessListRule URLs

Now add the URL includes for `AccessListRule` by extending `urlpatterns`:

```python
urlpatterns = (
    # Access lists
    path(
        'access-lists/',
        include(get_model_urls('netbox_access_lists', 'accesslist', detail=False)),
    ),
    path(
        'access-lists/<int:pk>/',
        include(get_model_urls('netbox_access_lists', 'accesslist')),
    ),

    # Access list rules
    path(
        'rules/',
        include(get_model_urls('netbox_access_lists', 'accesslistrule', detail=False)),
    ),
    path(
        'rules/<int:pk>/',
        include(get_model_urls('netbox_access_lists', 'accesslistrule')),
    ),
)
```

For reference, your plugin project should now include `urls.py`:

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
│   ├── urls.py
│   └── views.py
├── pyproject.toml
└── README.md
```

## Manually map views to URLs

:warning: **Warning:** This section is only required if you prefer to define URL patterns manually. If you are using `get_model_urls()`, you can skip ahead to [Test the Views](#test-the-views).

If you want manual routing, use a `urls.py` that maps each path to a view explicitly.

Open `urls.py` and replace its contents with the following.

URL mapping for NetBox plugins is pretty much identical to regular Django apps.
We define `urlpatterns` as an iterable of `path()` calls, mapping URL fragments to view classes.

First, import Django's `path` function, along with our plugin `models` and `views` modules:

```python
from django.urls import path

from . import models, views
```

We have four views per model, but we need five paths for each.
This is because the add and edit operations are handled by the same view, but accessed via different URLs.
Along with the URL and view for each path, we also specify a `name` so we can reference URLs easily in code.

```python
urlpatterns = (
    path('access-lists/', views.AccessListListView.as_view(), name='accesslist_list'),
    path('access-lists/add/', views.AccessListEditView.as_view(), name='accesslist_add'),
    path('access-lists/<int:pk>/', views.AccessListView.as_view(), name='accesslist'),
    path('access-lists/<int:pk>/edit/', views.AccessListEditView.as_view(), name='accesslist_edit'),
    path('access-lists/<int:pk>/delete/', views.AccessListDeleteView.as_view(), name='accesslist_delete'),
)
```

We chose `access-lists` as the base URL for our `AccessList` model, but you are free to choose something else.
However, it is recommended to retain the naming scheme shown, as several NetBox features rely on it.

:green_circle: **Tip:** When passing a class based view to `path()`, make sure you call `as_view()`.

Now add the rest of the paths.
You might find it helpful to group paths by model to keep the file easy to scan.

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

### Adding changelog views manually

NetBox supports automatic [change logging](https://netboxlabs.com/docs/netbox/features/change-logging/).
You can see this in the Changelog tab on many object detail pages.

If you are using the decorator-based approach, these extra views are often handled for you.
If you are mapping URLs manually, you can add changelog routes using `ObjectChangeLogView`.

First, import it:

```python
from netbox.views.generic import ObjectChangeLogView
```

Then add an extra path for each model:

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

Notice that we use `ObjectChangeLogView` directly here.
We do not need model-specific subclasses for it.
We pass the `model` kwarg so the view knows which model to work with.

## Test the Views

Now for the moment of truth: do the views show up in the UI?

Make sure the development server is running, then open a browser and navigate to:

```text
http://localhost:8000/plugins/access-lists/access-lists/
```

You should see the access list list view and, if you created objects in Step 2, an access list named `MyACL1`.

The first `access lists` part in the URL comes from the plugin `base_url` you defined in `__init__.py`.
The second part comes from the paths you defined in `urls.py`.

:blue_square: **Note:** This guide assumes you are running the Django development server locally on port 8000. If your setup differs, adjust the URL accordingly.

![Access lists list view](/images/step06-accesslist-list.png)

You should see the `name`, `rule_count`, and `default_action` columns we defined in Step 3.
The `rule_count` value should show two rules if you followed the example data creation in Step 2.

If you click the Add button in the top right, you should reach the access list creation form:

![Access list creation form](/images/step06-accesslist-form.png)

If you click an access list name in the table, you will likely hit a `TemplateDoesNotExist` error.
That is expected at this point because we have not created the object templates yet.
We will fix that in the next step.

:blue_square: **Note:** You might also notice that adding a rule fails with a `NoReverseMatch` error. This happens because the dynamic form fields require REST API endpoints for plugin models. We will build the REST API pieces later in the tutorial.

<div align="center">

:arrow_left: [Step 5: Views](/tutorial/step05-views.md) | [Step 7: Templates](/tutorial/step07-templates.md) :arrow_right:

</div>

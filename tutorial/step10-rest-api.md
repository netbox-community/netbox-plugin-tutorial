# Step 10: REST API

The REST API enables powerful integrations with other systems that exchange data with NetBox.

NetBox's API is built on the [Django REST Framework](https://www.django-rest-framework.org/) (DRF).
DRF is not part of Django itself, but it works very well with Django models and querysets.
In this step, we will extend NetBox's REST API to serve our plugin models.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step09-filter-sets` (if you cloned the repository `netbox-plugin-demo`).

Our API code will live in an `api/` package inside `netbox_access_lists/`.
Create the directory and an `__init__.py` file:

```bash
cd netbox_access_lists/
mkdir api
touch api/__init__.py
```

## Create Model Serializers

Serializers are somewhat analogous to forms.
A form renders HTML and validates user input from the UI.
A serializer defines how a model is represented in the API (usually as JSON), and how input data from the API is validated and converted into Python objects.

We will create one serializer per model.

Begin by creating `serializers.py` in the `api/` directory:

```bash
touch api/serializers.py
```

At the top of this file, import:

* NetBox's `NetBoxModelSerializer` base class
* DRF's `serializers` module (we will use it for fields like `HyperlinkedIdentityField`)
* NetBox's `PrefixSerializer` so we can include nested prefix data
* our plugin models

```python
from ipam.api.serializers import PrefixSerializer
from netbox.api.serializers import NetBoxModelSerializer
from rest_framework import serializers

from ..models import AccessList, AccessListRule
```

### Create `AccessListSerializer`

Create a serializer for `AccessList` by subclassing `NetBoxModelSerializer`.
Much like a model form, we define a `Meta` class that tells the serializer which model it is for, and which fields to include.

Start with the URL field.
Every serializer should include a read only `url` field that points to the API endpoint for this object.
We create this using DRF's `HyperlinkedIdentityField`.

```python
class AccessListSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslist-detail'
    )
```

The `view_name` here is the name of the API view that will serve a single access list object.
We will create it later when we register our API viewsets and URLs.

Next, remember the `rule_count` column we added for the table in Step 3. We can expose that value in the API too.
This is a computed field, so we make it read-only, and we will annotate it in the view queryset.

```python
    rule_count = serializers.IntegerField(read_only=True)
```

Now add the `Meta` class:

```python
    class Meta:
        model = AccessList
        fields = (
            'id',
            'url',
            'display',
            'name',
            'default_action',
            'rule_count',
            'comments',
            'tags',
            'custom_fields',
            'created',
            'last_updated',
        )
```

A quick overview of some common fields:

* `id` is the primary key. This is a must-have for every serializer.
* `display` is provided by `NetBoxModelSerializer`. It is read-only, and returns a human-friendly string representation of the object.
* `tags` and `custom_fields` are provided by NetBox's model and serializer helpers.
* `created` and `last_updated` come from `NetBoxModel` and are read-only.

:green_circle: **Tip:** The order in `fields` is the order you will see in the API response. Many NetBox serializers put `tags`, `custom_fields`, `created`, and `last_updated` near the end.

### Create `AccessListRuleSerializer`

Now create a serializer for `AccessListRule`.
This will look similar, but it includes several foreign keys and list fields.

Start with the URL field again:

```python
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail'
    )
```

Now define the `Meta` class. Make sure we include `description`, since it is part of the model and useful for API consumers.

```python
    class Meta:
        model = AccessListRule
        fields = (
            'id',
            'url',
            'display',
            'access_list',
            'index',
            'protocol',
            'source_prefix',
            'source_ports',
            'destination_prefix',
            'destination_ports',
            'action',
            'description',
            'comments',
            'tags',
            'custom_fields',
            'created',
            'last_updated',
        )
```

#### Nested serializers for related objects

By default, serializers often represent related objects by primary key only.
That works, but it forces clients to make extra requests just to display basic info like a name.

NetBox supports nested representations for related objects, which usually include an `id`, a `url`, and a `display` value.
This is very convenient for both humans and integrations.

For `source_prefix` and `destination_prefix`, we can use NetBox's nested prefix serializer:

```python
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail'
    )
    source_prefix = PrefixSerializer(nested=True, required=False, allow_null=True)
    destination_prefix = PrefixSerializer(nested=True, required=False, allow_null=True)
```

We also want a nested representation for `access_list`.
To do that, we will make sure our `AccessListSerializer` defines `brief_fields`, and then reuse it with `nested=True`.

### Enable nested representations for plugin models

`NetBoxModelSerializer` supports nested mode using `brief_fields`.
These fields define what should be included when the serializer is used with `nested=True`.

Update both serializer `Meta` classes to include `brief_fields`:

```python
class AccessListSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslist-detail'
    )
    rule_count = serializers.IntegerField(read_only=True)

    class Meta:
        model = AccessList
        fields = (
            'id',
            'url',
            'display',
            'name',
            'default_action',
            'rule_count',
            'comments',
            'tags',
            'custom_fields',
            'created',
            'last_updated',
        )
        brief_fields = ('id', 'url', 'display', 'name')


class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail'
    )
    access_list = AccessListSerializer(nested=True)
    source_prefix = PrefixSerializer(nested=True, required=False, allow_null=True)
    destination_prefix = PrefixSerializer(nested=True, required=False, allow_null=True)

    class Meta:
        model = AccessListRule
        fields = (
            'id',
            'url',
            'display',
            'access_list',
            'index',
            'protocol',
            'source_prefix',
            'source_ports',
            'destination_prefix',
            'destination_ports',
            'action',
            'description',
            'comments',
            'tags',
            'custom_fields',
            'created',
            'last_updated',
        )
        brief_fields = ('id', 'url', 'display', 'index')
```

:green_circle: **Tip:** It is good practice to include `id`, `url`, and `display` in `brief_fields`.

For reference, your plugin project should now include `api/serializers.py`:

```text
.
├── netbox_access_lists
│   ├── api
│   │   ├── __init__.py
│   │   └── serializers.py
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

## Create the API viewsets

Next, we need API views.
In the UI, we created multiple views per model (detail, list, edit, delete).
For the REST API, DRF uses viewsets.
A viewset is a single class that can handle the common API operations for a model, like listing objects, retrieving one object, creating, updating, and deleting.

Create `api/views.py`:

```bash
touch api/views.py
```

Add the imports below:

```python
from django.db.models import Count
from netbox.api.viewsets import NetBoxModelViewSet

from .. import filtersets, models
from .serializers import AccessListSerializer, AccessListRuleSerializer
```

### AccessList viewset

Create a viewset for access lists. We set:

* `queryset` to the objects we want to expose
* `serializer_class` to the serializer we created earlier

We also annotate `rule_count` so the serializer field is populated.

```python
class AccessListViewSet(NetBoxModelViewSet):
    queryset = models.AccessList.objects.prefetch_related('tags').annotate(
        rule_count=Count('rules')
    )
    serializer_class = AccessListSerializer
```

### AccessListRule viewset

Now create a viewset for rules.
Here we also connect the filter set from Step 9 using `filterset_class`.
This is what allows the list view endpoint to support query filtering.

For performance, we load foreign key relationships using `select_related`, and tags using `prefetch_related`.

```python
class AccessListRuleViewSet(NetBoxModelViewSet):
    queryset = models.AccessListRule.objects.select_related(
        'access_list', 'source_prefix', 'destination_prefix'
    ).prefetch_related('tags')
    serializer_class = AccessListRuleSerializer
    filterset_class = filtersets.AccessListRuleFilterSet
```

## Create the endpoint URLs

Finally, we need to expose these viewsets under API URLs.
This works a bit differently from UI URLs.
Instead of defining many `path()` entries, we register each viewset with a router.

Create `api/urls.py`:

```bash
touch api/urls.py
```

Add the imports:

```python
from netbox.api.routers import NetBoxRouter

from . import views
```

Set `app_name`.
NetBox uses this namespace to resolve view names, including the `view_name` values we used in our serializers.

```python
app_name = 'netbox_access_lists-api'
```

Now create a router and register each viewset:

```python
router = NetBoxRouter()
router.register('access-lists', views.AccessListViewSet)
router.register('access-list-rules', views.AccessListRuleViewSet)

urlpatterns = router.urls
```

:green_circle: **Tip:** The base URL for plugin API endpoints is determined by the `base_url` in your plugin config (Step 1). In our case, the API root is under `/api/plugins/access-lists/`.

## Test the API

With all REST API components in place, you should be able to browse the plugin API endpoints.

While logged into NetBox, open:

<http://localhost:8000/api/plugins/access-lists/>

You should see the available endpoints.
Clicking on an endpoint will show a list view, and from there you can click through to individual objects.

:blue_square: **Note:** If the REST API endpoints do not load, restart the development server (`manage.py runserver`) and refresh the page.

![REST API root view](/images/step10-rest-api1.png)

![REST API access list rules](/images/step10-rest-api2.png)

For reference, your plugin project should now include the full `api/` directory:

```text
.
├── netbox_access_lists
│   ├── api
│   │   ├── __init__.py
│   │   ├── serializers.py
│   │   ├── urls.py
│   │   └── views.py
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

<div align="center">

:arrow_left: [Step 9: Filter Sets](/tutorial/step09-filter-sets.md) | [Step 11: GraphQL API](/tutorial/step11-graphql-api.md) :arrow_right:

</div>

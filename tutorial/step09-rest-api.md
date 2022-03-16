# Step 9: REST API

The REST API enables powerful integration with other systems which exchange data with NetBox. It is powered by the [Django REST Framework](https://www.django-rest-framework.org/) or DRF (which is _not_ a component of Django itself). In this tutorial, we'll see how we can extend NetBox's REST API to serve our plugin.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step08-filter-sets`.

Our API code will live in the `api/` directory under `netbox_access_lists/`. Let's go ahead and create that as well as an `__init__.py` now:

```bash
$ cd netbox_access_lists/
$ mkdir api
$ touch api/__init__.py
```

## Create Model Serializers

Serializers are somewhat analogous to forms: They control the translation of client data to and from Python objects, while Django itself handles the database abstraction. We need to create a serializer for each of our models. Begin by creating `serializers.py` in the `api/` directory.

```bash
$ edit api/serializers.py
```

At the top of this file, we need to import the `serializers` module from the `rest_framework` library, as well as NetBox's `NetBoxModelSerializer` class and our plugin's own models:

```python
from rest_framework import serializers

from netbox.api.serializers import NetBoxModelSerializer
from ..models import AccessList, AccessListRule
```

### Create AccessListSerializer

First, we'll create a serializer for `AccessList`, subclassing `NetBoxModelSerializer`. Much like when creating a model form, we'll create a child `Meta` class under the serializer specifying the associated `model` and the `fields` to be included.

```python
class AccessListSerializer(NetBoxModelSerializer):

    class Meta:
        model = AccessList
        fields = (
            'id', 'display', 'name', 'default_action', 'comments', 'tags', 'custom_fields', 'created',
            'last_updated',
        )
```

It's worth discussing each of the fields we've named above. `id` is the model's primary key; it should always be included with every model, as it provides a guaranteed method of uniquely identifying objects. The `display` field is built into `NetBoxModelSerializer`: It is a read-only field which returns a string representation of the object. This is useful for populating form field dropdowns, for instance.

The `name`, `default_action`, and `comments` fields are declared on the `AccessList` model. `tags` provides access to the object's tag manager, and `custom_fields` provides access to its custom field data; both of these are provided by `NetBoxModelSerializer`. Finally, the `created` and `last_updated` are read-only fields built into `NetBoxModel`.

Our serializer will introspect the model to generate the necessary fields automatically, however there's one field that we need to add manually. Every serializer should include a read-only `url` field which contains the URL where the object can be reached; think of it as similar to a model's `get_absolute_url()` method. To add this, we'll use DRF's `HyperlinkedIdentityField`. Add it above the `Meta` child class:

```python
class AccessListSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslist-detail'
    )
```

When invoking the field class, we need to specify the appropriate view name. Note that this view doesn't actually exist yet; we'll create it in the next section.

We also need to add the `url` field to `Meta.fields`:

```python
    class Meta:
        model = AccessList
        fields = (
            'id', 'url', 'display', 'name', 'default_action', 'comments', 'tags', 'custom_fields', 'created',
            'last_updated',
        )
```

Remember back in step three when we added a table column showing the number of rules assigned to each access list? That was handy. Let's add a serializer field for it too! Add this directly below the `url` field:

```python
rule_count = serializers.IntegerField(read_only=True)
```

Just as with the table column, we'll rely on our view (to be defined next) to annotate the rule count for each access list on the underlying queryset.

Finally, we need to add `rule_count` to `Meta.fields`:

```python
    class Meta:
        model = AccessList
        fields = (
            'id', 'url', 'display', 'name', 'default_action', 'comments', 'tags', 'custom_fields', 'created',
            'last_updated', 'rule_count',
        )
```

:green_circle: **Tip:** The order in which fields are listed determines the order in which they appear in the object's API representation.

### Create AccessListRuleSerializer

We also need to create a serializer for `AccessListRule`. Add it to `serializers.py` below `AccessListSerializer`. As with the first serializer, we'll add a `Meta` class to define the model and fields, and a `url` field.

```python
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail')

    class Meta:
        model = AccessListRule
        fields = (
            'id', 'url', 'display', 'access_list', 'index', 'protocol', 'source_prefix', 'source_ports',
            'destination_prefix', 'destination_ports', 'action', 'tags', 'custom_fields', 'created', 'last_updated',
        )
```

There's an additional consideration when referencing related objects in a serializer. By default, the serializer will return only the primary key of the related object; its numeric ID. This requires the client to make additional API requests in order to determine _any_ other information about the related object. It is convenient to provide some information about the related object, such as its name and URL, automatically. We can do this by using a _nested serializer_.

For instance, the `source_prefix` and `destination_prefix` fields both reference NetBox's core `ipam.Prefix` model. We can extend `AccessListRuleSerializer` to use NetBox's nested serializer for this model:

```python
from ipam.api.serializers import NestedPrefixSerializer
# ...
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail')
    source_prefix = NestedPrefixSerializer()
    destination_prefix = NestedPrefixSerializer()
```

Now, our serializer will include an abridged representation of the source and/or destination prefixes for the object. We should do this with the `access_list` field as well, however we'll first need to create a nested serializer for the `AccessList` model.

### Create Nested Serializers

Begin by importing NetBox's `WritableNestedSerializer` class. This will serve as the base class for our nested serializers.

```python
from netbox.api.serializers import NetBoxModelSerializer, WritableNestedSerializer
```

Then, create two nested serializer classes, one for each of our plugin's models. Each of these will have a `url` field and `Meta` child class like the regular serializers, however the `Meta.fields` attribute for each is limited to a bare minimum of fields: `id`, `url`, `display`, and a supplementary human-friendly identifier. Add these in `serializers.py` _above_ the regular serializers (because we need to define `NestedAccessListSerializer` before we can reference it). 

```python
class NestedAccessListSerializer(WritableNestedSerializer):
    url = serializers.HyperlinkedIdentityField(view_name='plugins-api:netbox_access_lists-api:accesslist-detail')

    class Meta:
        model = AccessList
        fields = ('id', 'url', 'display', 'name')

class NestedAccessListRuleSerializer(WritableNestedSerializer):
    url = serializers.HyperlinkedIdentityField(view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail')

    class Meta:
        model = AccessListRule
        fields = ('id', 'url', 'display', 'index')
```

Now we can override the `access_list` field on `AccessListRuleSerializer` to use the nested serializer:

```python
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail')
    access_list = NestedAccessListSerializer()
    source_prefix = NestedPrefixSerializer()
    destination_prefix = NestedPrefixSerializer()
```

## Create the Views

Next, we need to create views to handle the API logic. Just as serializers are roughly analogous to forms, API views work similarly to the UI views that we created in five. However, because API functionality is highly standardized, view creation is substantially simpler: We generally need only to create a single _view set_ for each model. A single view set can handle the view, add, change, and delete operations which required dedicated UI views.

Start by creating `api/views.py` and importing NetBox's `NetBoxModelViewSet` class, as well as our plugin's `models` and `filtersets` modules, and our serializers.

```python
from netbox.api.viewsets import NetBoxModelViewSet

from .. import filtersets, models
from .serializers import AccessListSerializer, AccessListRuleSerializer
```

First we'll create a view set for access lists, by inheriting from `NetBoxModelViewSet` and defining its `queryset` and `serializer_class` attributes. (Note that we're prefetching assigned tags for the queryset.)

```python
class AccessListViewSet(NetBoxModelViewSet):
    queryset = models.AccessList.objects.prefetch_related('tags')
    serializer_class = AccessListSerializer
```

Recall that we added a `rule_count` field to `AccessListSerializer`; let's annotate the queryset appropriately to ensure that field gets populated (just as we did for the table column in step five). Remember to import Django's `Count` utility class.

```python
from django.db.models import Count
# ...
class AccessListViewSet(NetBoxModelViewSet):
    queryset = models.AccessList.objects.prefetch_related('tags').annotate(
        rule_count=Count('rules')
    )
    serializer_class = AccessListSerializer
```

Next, we'll add a view set for access list rules. In addition to `queryset` and `serializer_class`, we'll attach the filter set for this model as `filterset_class`. Note that we're also prefetching all related object fields in addition to tags to improve performance when listing many objects.

```python
class AccessListRuleViewSet(NetBoxModelViewSet):
    queryset = models.AccessListRule.objects.prefetch_related(
        'access_list', 'source_prefix', 'destination_prefix', 'tags'
    )
    serializer_class = AccessListRuleSerializer
    filterset_class = filtersets.AccessListRuleFilterSet
```

## Create the Endpoint URLs

Finally, we'll create our API endpoint URLs. This works a bit differently from UI views: Instead of defining a series of paths, we instantiate a router and register each view set with it.

Create `api/urls.py` and import NetBox's `NetBoxRouter` and our API views:

```python
from netbox.api.routers import NetBoxRouter
from . import views
```

Next, we'll define an `app_name`: This will be used to resolve API view names for our plugin.

```python
app_name = 'netbox_access_list'
```

Then, we create an `OrderedDefaultRouter` instance and register each view with it using our desired URL. These are the endpoints that will be available under `/api/plugins/access-lists/`.

```python
router = NetBoxRouter()
router.register('access-lists', views.AccessListViewSet)
router.register('access-list-rules', views.AccessListRuleViewSet)
```

Finally, we expose the router's `urls` attribute as `urlpatterns` so that it will be detected by the plugins framework.

:green_circle: **Tip:** The base URL for our plugin's REST API endpoints is determined by the `base_url` attribute of the plugin config class that we created in step one.

With all of our REST API components now in place, we should be able to make API requests. (Note that you'll need to provision a token first.) You can quickly verify that our endpoints are working properly by navigating to <http://localhost:8000/api/plugins/access-lists/> in your browser while logged into NetBox. You should see the two available endpoints; clicking on either will return a list of objects.

![REST API - Root view](/images/step09-rest-api1.png)

![REST API - Access list rules](/images/step09-rest-api2.png)

:arrow_right: [Step 10: GraphQL API](/tutorial/step10-graphql-api.md)


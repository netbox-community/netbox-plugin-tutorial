# Step 10: GraphQL API

In addition to its REST API, NetBox also features a [GraphQL](https://graphql.org/) API. This can be used to conveniently request arbitrary collections of data about NetBox objects. NetBox's GraphQL API is built using the [Graphene](https://graphene-python.org/) and [`graphene-django`](https://docs.graphene-python.org/projects/django/en/latest/) library.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step09-rest-api`.

Begin by creating `graphql.py`. This will hold our object type and query classes.

```bash
$ cd netbox_access_lists/
$ edit graphql.py
```

We'll need to import several resources. First we need Graphene's `ObjectType` class, as well as NetBox's custom `NetBoxObjectType` which inherits from it. (The latter will be used for our model types.) We also need the `ObjectField` and `ObjectListField` classes provided by NetBox for our query. And finally, import our plugin's `models` and `filtersets` modules.

```python
from graphene import ObjectType
from netbox.graphql.types import NetBoxObjectType
from netbox.graphql.fields import ObjectField, ObjectListField
from . import filtersets, models
```

## Create the Object Types

Subclass `NetBoxObjectType` to create two object type classes, one for each of our models. Just like with the REST API serilizers, create a child `Meta` class on each defining its `model` and `fields`. However, instead of explicitly listing each field by name, in our case we can use the special value `__all__` to indicate that we want to include all available model fields. Additionally, declare `filterset_class` on `AccessListRuleType` to attach the filter set.

```python
class AccessListType(NetBoxObjectType):

    class Meta:
        model = models.AccessList
        fields = '__all__'


class AccessListRuleType(NetBoxObjectType):

    class Meta:
        model = models.AccessListRule
        fields = '__all__'
        filterset_class = filtersets.AccessListRuleFilterSet
```

## Create the Query

Then we need to create our query class. Subclass Graphene's `ObjectType` class and define two fields for each model: an object field and a list field.

```python
class Query(ObjectType):
    access_list = ObjectField(AccessListType)
    access_list_list = ObjectListField(AccessListType)

    access_list_rule = ObjectField(AccessListRuleType)
    access_list_rule_list = ObjectListField(AccessListRuleType)
```

Then we just need to expose our query class to the plugins framework as `schema`:

```python
schema = Query
```

:green_circle: **Tip:** The path to the query class can be changed by setting `graphql_schema` in the plugin's configuration class.

To try out the GraphQL API, open `<http://netbox:8000/graphql/>` in a browser and enter the following query:

```
query {
  access_list_list {
    id
    name
      rules {
      index
      action
      description
    }
  }
}
```

You should receive a response showing the ID, name, and rules for each access list in NetBox. Each rule will list its index, action, and description. Experiment with different queries to see what other data you can request. (Refer back to the model definitions for inspiration.)

![GraphiQL interface](/images/step10-graphiql.png)

This completes the plugin development tutorial. Well done! Now you're all set to make a plugin of your own!

<div align="center">

:arrow_left: [Step 9: REST API](/tutorial/step09-rest-api.md) | [Step 11: Search](/tutorial/step11-search.md) :arrow_right:

</div>


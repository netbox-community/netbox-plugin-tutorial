# Step 11: GraphQL API

In addition to its REST API, NetBox also provides a [GraphQL](https://graphql.org/) API.
GraphQL is useful when you want to request exactly the fields you need, including related objects, in a single query.

NetBox builds its GraphQL API using the [Strawberry GraphQL](https://github.com/strawberry-graphql/strawberry) and [Strawberry Django](https://github.com/strawberry-graphql/strawberry-django) libraries.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step10-rest-api` (in case you've cloned the repository `netbox-plugin-demo`).

Our GraphQL code will live in a `graphql/` package inside `netbox_access_lists/`.
Create the directory and an `__init__.py` file:

```bash
cd netbox_access_lists/
mkdir graphql
touch graphql/__init__.py
```

## Create enum types

GraphQL requires enums for fields that have a fixed set of allowed values.
In our plugin, we already have fixed choices defined as ChoiceSets:

* `ActionChoices` for `AccessList.default_action` and `AccessListRule.action`
* `ProtocolChoices` for `AccessListRule.protocol`

NetBox ChoiceSets provide a helper to convert them into Python Enum classes, and Strawberry can convert those Enum classes into GraphQL enums.

Create `graphql/enums.py`:

```bash
touch graphql/enums.py
```

Add the imports and enum definitions:

```python
import strawberry

from ..choices import ActionChoices, ProtocolChoices


ActionEnum = strawberry.enum(ActionChoices.as_enum())
ProtocolEnum = strawberry.enum(ProtocolChoices.as_enum())
```

## Create GraphQL filters

For REST API filtering, we used `FilterSet` classes from Step 9.
GraphQL filtering is separate, so we create dedicated GraphQL filter classes.

NetBox provides `NetBoxModelFilter` to make this easier.

Create `graphql/filters.py`:

```bash
touch graphql/filters.py
```

Add the imports:

```python
from typing import TYPE_CHECKING, Annotated

import strawberry
import strawberry_django
from netbox.graphql.filters import NetBoxModelFilter
from strawberry import ID
from strawberry_django import FilterLookup

from .. import models
```

### Why `TYPE_CHECKING` and `strawberry.lazy`

In GraphQL schemas it is common for types and filters to reference each other.
If we import everything normally, we can end up with circular imports.

To avoid that, we do two things:

* import types only inside an `if TYPE_CHECKING:` block so they are available to type checkers
* reference those types using `strawberry.lazy(...)` so Strawberry can resolve them later

Add this below the imports:

```python
if TYPE_CHECKING:
    from ipam.graphql.filters import PrefixFilter
    from netbox.graphql.filter_lookups import IntegerArrayLookup, IntegerLookup

    from .enums import ActionEnum, ProtocolEnum
```

### AccessListFilter

Create a filter class for `AccessList`.
The `@strawberry_django.filter(...)` decorator connects the filter class to the model.

```python
@strawberry_django.filter_type(models.AccessList, lookups=True)
class AccessListFilter(NetBoxModelFilter):
    name: FilterLookup[str] | None = strawberry_django.filter_field()
    default_action: (
        Annotated['ActionEnum', strawberry.lazy('netbox_access_lists.graphql.enums')]
        | None
    ) = strawberry_django.filter_field()
```

A few notes:

* `lookups=True` enables lookup operators like `exact`, `icontains`, `in`, and so on, depending on the field type.
* `FilterLookup[str]` means the field supports string lookups.
* `default_action` is an enum value, so we annotate it with `ActionEnum`.

### AccessListRuleFilter

Now create a filter class for `AccessListRule`.
This includes:

* a nested filter for `access_list`
* ID based filters for related objects (useful and common)
* integer lookups for `index`
* integer array lookups for ports
* prefix filters for source and destination prefixes
* enum filters for protocol and action

Add this below `AccessListFilter`:

```python
@strawberry_django.filter_type(models.AccessListRule, lookups=True)
class AccessListRuleFilter(NetBoxModelFilter):
    access_list: (
        Annotated[
            'AccessListFilter', strawberry.lazy('netbox_access_lists.graphql.filters')
        ]
        | None
    ) = strawberry_django.filter_field()
    access_list_id: ID | None = strawberry_django.filter_field()
    index: (
        Annotated['IntegerLookup', strawberry.lazy('netbox.graphql.filter_lookups')]
        | None
    ) = strawberry_django.filter_field()
    protocol: (
        Annotated['ProtocolEnum', strawberry.lazy('netbox_access_lists.graphql.enums')]
        | None
    ) = strawberry_django.filter_field()
    source_prefix: (
        Annotated['PrefixFilter', strawberry.lazy('ipam.graphql.filters')] | None
    ) = strawberry_django.filter_field()
    source_prefix_id: ID | None = strawberry_django.filter_field()
    source_ports: (
        Annotated[
            'IntegerArrayLookup', strawberry.lazy('netbox.graphql.filter_lookups')
        ]
        | None
    ) = strawberry_django.filter_field()
    destination_prefix: (
        Annotated['PrefixFilter', strawberry.lazy('ipam.graphql.filters')] | None
    ) = strawberry_django.filter_field()
    destination_prefix_id: ID | None = strawberry_django.filter_field()
    destination_ports: (
        Annotated[
            'IntegerArrayLookup', strawberry.lazy('netbox.graphql.filter_lookups')
        ]
        | None
    ) = strawberry_django.filter_field()
    action: (
        Annotated['ActionEnum', strawberry.lazy('netbox_access_lists.graphql.enums')]
        | None
    ) = strawberry_django.filter_field()
```

At this point, `graphql/filters.py` is complete.

## Create the object types

GraphQL needs object types to describe what fields are available and how models relate to each other.
NetBox provides `NetBoxObjectType` as a convenient base class.

Create `graphql/types.py`:

```bash
touch graphql/types.py
```

Add the imports:

```python
from typing import TYPE_CHECKING, Annotated, List

import strawberry
import strawberry_django
from netbox.graphql.types import NetBoxObjectType

from .. import models
from . import filters
```

For prefix fields, we reference the existing type from NetBox IPAM:

```python
if TYPE_CHECKING:
    from ipam.graphql.types import PrefixType
```

Now define types for both models.

We use `fields="__all__"` to include all model fields automatically, and we connect the GraphQL filters using `filters=...`.

```python
@strawberry_django.type(
    models.AccessList, fields='__all__', filters=filters.AccessListFilter
)
class AccessListType(NetBoxObjectType):
    # Related models
    rules: List[
        Annotated[
            'AccessListRuleType', strawberry.lazy('netbox_access_lists.graphql.types')
        ]
    ]


@strawberry_django.type(
    models.AccessListRule, fields='__all__', filters=filters.AccessListRuleFilter
)
class AccessListRuleType(NetBoxObjectType):
    # Model fields
    access_list: Annotated[
        'AccessListType', strawberry.lazy('netbox_access_lists.graphql.types')
    ]
    source_prefix: Annotated['PrefixType', strawberry.lazy('ipam.graphql.types')] | None
    destination_prefix: (
        Annotated['PrefixType', strawberry.lazy('ipam.graphql.types')] | None
    )
```

A few notes:

* `AccessListType.rules` exposes the reverse foreign key relationship.
* `AccessListRuleType.access_list` exposes the forward foreign key relationship.
* `source_prefix` and `destination_prefix` reuse NetBox core GraphQL types.
* Both prefix fields are optional because they are nullable in the model.

## Create the query

Now we need to expose our types through GraphQL query fields.
NetBox will merge plugin query classes into the main schema.

Create `graphql/schema.py`:

```bash
touch graphql/schema.py
```

Add the imports:

```python
import strawberry
import strawberry_django

from .types import AccessListType, AccessListRuleType
```

Now define a query class.
We will provide a single object field and a list field for each model.

```python
@strawberry.type(name='Query')
class NetBoxAccessListQuery:
    access_list: AccessListType = strawberry_django.field()
    access_list_list: list[AccessListType] = strawberry_django.field()

    access_list_rule: AccessListRuleType = strawberry_django.field()
    access_list_rule_list: list[AccessListRuleType] = strawberry_django.field()
```

In general:

* the single object fields let you fetch one object, typically by ID
* the list fields let you fetch collections and can support filtering

## Register the schema with the plugin

Finally, expose your query class from `graphql/__init__.py`.

Open `graphql/__init__.py`:

```bash
touch graphql/__init__.py
```

Add:

```python
from .schema import NetBoxAccessListQuery

schema = [NetBoxAccessListQuery]
```

:green_circle: **Tip:** The plugin config can also control schema discovery using the `graphql_schema` setting. See the NetBox plugin GraphQL documentation for details: [GraphQL API](https://netboxlabs.com/docs/netbox/plugins/development/graphql-api/).

## Test the GraphQL API

To try out the GraphQL API, open `<http://localhost:8000/graphql/>` in a browser.
In the query editor, enter:

```graphql
query {
  access_list_list {
    id
    name
    rules {
      index
      description
      protocol
      action
    }
  }
}
```

You should receive a response showing the ID, name, and rules for each access list.
Try adding or removing fields to see how the response changes.
If you are unsure what fields exist, refer back to the model definitions.

![GraphiQL interface](/images/step11-graphiql.png)

For reference, your plugin project should now include the `graphql/` package:

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
│   ├── graphql
│   │   ├── enums.py
│   │   ├── filters.py
│   │   ├── __init__.py
│   │   ├── schema.py
│   │   └── types.py
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

:arrow_left: [Step 10: REST API](/tutorial/step10-rest-api.md) | [Step 12: Search](/tutorial/step12-search.md) :arrow_right:

</div>

# Step 12: Global Search

In this step, we will add search index classes for our plugin models so they appear in NetBox global search results.

NetBox global search is meant to help users quickly find objects by typing a few keywords.
Under the hood, NetBox builds a small search index for each model that supports it.
Each index tells NetBox which fields to cache for search and how important each field is.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step10-graphql` (if you cloned the repository `netbox-plugin-demo`).

## Create search indexes

Our plugin has two models: `AccessList` and `AccessListRule`.
We want users to be able to find both via global search.
To do that, we need to declare and register a `SearchIndex` subclass for each model.

Begin by creating `search.py` in the plugin package root, alongside `models.py`.

```bash
cd netbox_access_lists/
touch search.py
```

### Define a basic index class per model

Start by importing `SearchIndex` and our models.
Then create a basic index class for each model:

```python
from netbox.search import SearchIndex

from .models import AccessList, AccessListRule


class AccessListIndex(SearchIndex):
    model = AccessList


class AccessListRuleIndex(SearchIndex):
    model = AccessListRule
```

This tells NetBox which model each index belongs to.
Next, we will tell NetBox which fields should be searchable.

### Choose fields and weights

Each index can declare a `fields` attribute, which is a tuple of `(field_name, weight)` pairs.

Weights control how strongly a match influences ranking.
In general:

* lower numbers mean higher priority in the results
* higher numbers mean lower priority

#### AccessList fields

Our `AccessList` model has these interesting fields:

* `name` is the primary identifier, so we want it to rank highly
* `default_action` is useful for filtering, but it is not usually something people search for
* `comments` can contain helpful keywords, but matches there should rank lower than matches on `name`

A reasonable starting point is:

* `name` with weight `100`
* `comments` with weight `5000`

```python
class AccessListIndex(SearchIndex):
    model = AccessList
    fields = (
        ('name', 100),
        ('comments', 5000),
    )
```

#### AccessListRule fields

For `AccessListRule`, the most useful free text field is `description`.
We will index that:

```python
class AccessListRuleIndex(SearchIndex):
    model = AccessListRule
    fields = (
        ('description', 500),
    )
```

You might wonder why we are not indexing source and destination prefixes or ports.

* Prefixes are related objects. Indexing related values can lead to stale results if the related object changes.
* Ports are common numbers, and indexing them tends to produce noisy results.
* These fields are usually better handled by filters (including the filter set we built in Step 9).

### Show more context in search results

Even if we do not index some fields, it can still be helpful to show them in the search result preview.
We can do that using `display_attrs`.

`display_attrs` controls which attributes are shown to the user alongside the result.
It does not change which fields are searched, it only affects what is displayed.

Add `display_attrs` like this:

```python
class AccessListIndex(SearchIndex):
    model = AccessList
    fields = (
        ('name', 100),
        ('comments', 5000),
    )
    display_attrs = ('name', 'default_action')


class AccessListRuleIndex(SearchIndex):
    model = AccessListRule
    fields = (
        ('description', 500),
    )
    display_attrs = (
        'access_list',
        'index',
        'protocol',
        'source_prefix',
        'source_ports',
        'destination_prefix',
        'destination_ports',
        'action',
        'description',
    )
```

## Register the index classes

To make NetBox aware of our search indexes, we need to register them using the `register_search` decorator.

Update `search.py` to import `register_search` and decorate both index classes:

```python
from netbox.search import SearchIndex, register_search

from .models import AccessList, AccessListRule


@register_search
class AccessListIndex(SearchIndex):
    model = AccessList
    fields = (
        ('name', 100),
        ('comments', 5000),
    )
    display_attrs = ('name', 'default_action')


@register_search
class AccessListRuleIndex(SearchIndex):
    model = AccessListRule
    fields = (
        ('description', 500),
    )
    display_attrs = (
        'access_list',
        'index',
        'protocol',
        'source_prefix',
        'source_ports',
        'destination_prefix',
        'destination_ports',
        'action',
        'description',
    )
```

## Build the search cache

With the index classes registered, we need to build the search cache for existing objects.
NetBox provides the `reindex` management command for this.

Run:

```shell
(venv) $ python netbox/manage.py reindex netbox_access_lists
Reindexing 2 models.
Clearing cached values... 0 entries deleted.
Indexing models
  netbox_access_lists.accesslist... 1 entries cached.
  netbox_access_lists.accesslistrule... 2 entries cached.
```

New objects created after this point are indexed automatically when they are saved.
You will usually only need to run `reindex` again if you change your index definitions or if you want to rebuild the cache for any reason.

Now try global search in the NetBox UI.
You should be able to find access lists and rules by searching for values in the indexed fields.

![Search results](/images/step12-search-results.png)

For reference, your plugin project should now include `search.py`:

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
│   ├── search.py
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

## Conclusion

This completes the plugin development tutorial. Nice work!

From here, a good next step is to review the NetBox plugin development documentation and start experimenting with a small plugin idea of your own.

<div align="center">

:arrow_left: [Step 11: GraphQL API](/tutorial/step11-graphql-api.md)

</div>

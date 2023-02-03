# Step 11: NetBox v3.4 Features

NetBox version 3.4, released in December 2022, introduced a greatly improved global search engine, which includes the ability for plugins to register their own models. In this step, we'll add search indexers for our custom models so that they appear in NetBox's global search results.

:warning: **Warning:** This feature requires NetBox v3.4 or later. If you haven't already, be sure to set `min_version = '3.4.0'` in `NetBoxAccessListsConfig`.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step10-graphql`.

## Create Search Indexes

Our plugin has two models: `AccessList` and `AccessListRule`. We'd like users to be able to search instances of both models using NetBox's global search feature. To enable this, we need to declare and register a `SearchIndex` subclass for each model.

Begin by creating `search.py` in the plugin's root directory, alongside `models.py`.

```bash
$ cd netbox_access_lists/
$ edit search.py
```

Within this file, we'll import NetBox's `SearchIndex` class as well as our own models. Then, we'll create a subclass of `SearchIndex` for each model:

```python
from netbox.search import SearchIndex
from .models import AccessList, AccessListRule

class AccessListIndex(SearchIndex):
    model = AccessList

class AccessListRuleIndex(SearchIndex):
    model = AccessListRule
```

There's a bit more to enabling search, though. We also need to tell NetBox which fields to search for each model, and how important each field is (also known as its _precedence_). The latter is accomplished by assigning a numerical weight.

Consider our `AccessList` model. It has three interesting database fields: `name`, `default_action`, and `comments`. How should we treat these when searching for objects in NetBox? This can be somewhat subjective, but generally we want to assign higher precedence (_lower_ weights) to important fields, and omit fields that we don't care about. If you're unsure what weights to assign, have a look around the core NetBox code base for similar examples.

* `name`: This is an important field, so we'll give it a high precedence of `100`.
* `default_action` This is a choice selection field. While very useful for _filtering_, we wouldn't typically expect users to search for these values. We'll exclude this field from the search index.
* `comments`: It's always recommended to include user comments in the search index, however we'll assign this field a much lower precedence of `5000` as any matches are less likely to be pertinent.

After selecting our search fields and their precedences, we should have something like this:

```python
class AccessListIndex(SearchIndex):
    model = AccessList
    fields = (
        ('name', 100),
        ('comments', 5000),
    )

class AccessListRuleIndex(SearchIndex):
    model = AccessListRule
    fields = (
        ('description', 500),
    )
```

Why did we exclude the source and destination parameters from `AccessListRuleIndex`? The source and destination prefixes are related objects, so we want to avoid caching their values locally: If the related object is changed, our cached copy can become outdated. And we omit the source and destination port numbers because matching on common integer values can produce a ton of irrelevant search results. All of these values are better matched using specific filters rather than general purpose search.

## Register the Indexers

Finally, we need to register our indexers so that NetBox knows to run them. At the top of `search.py`, import the `register_search` decorator. Then, use it to wrap both of our index classes:

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

@register_search
class AccessListRuleIndex(SearchIndex):
    model = AccessListRule
    fields = (
        ('description', 500),
    )
```

With our indexers now registered, we can run the `reindex` management command to index any existing objects. (New objects created from this point forward will be registered automatically upon creation.)

```
$ ./manage.py reindex netbox_access_lists
Reindexing 2 models.
Indexing models
  netbox_access_lists.accesslist... 1 entries cached.
  netbox_access_lists.accesslistrule... 3 entries cached.
```

Now we can search for access lists and rules using NetBox's global search function.

![Search results](/images/step11-search-results.png)

<div align="center">

:arrow_left: [Step 10: GraphQL API](/tutorial/step10-graphql-api.md)

</div>

# Step 2: Models

In this step, we will define some Django models to hold our plugin's data.

A model is a Python class that represents a table in the PostgreSQL database.
Each instance of a model class maps to a single row in that table.
We use models instead of raw SQL because working with Python objects is usually simpler, safer, and easier to evolve over time.

:blue_square: **Note:** If you skipped the previous step, and you cloned the `netbox-plugin-demo` repository, you can catch up by running `git checkout step01-plugin-configuration`.

## Create the Models

First, change into the `netbox_access_lists` directory and create a file named `models.py`.
This is where our model classes will live.

```bash
cd netbox_access_lists
touch models.py
```

Open `models.py` in your editor and add the imports below.

We import:

* Django's `models` library for defining fields
* NetBox's `NetBoxModel` base class so our models get NetBox features like custom fields, tags, and change logging
* PostgreSQL's `ArrayField`, which we will use to store a list of ports on a rule

```python
from django.contrib.postgres.fields import ArrayField
from django.db import models
from netbox.models import NetBoxModel
```

We will create two models:

* `AccessList` represents an access list, with a name and one or more rules
* `AccessListRule` represents a single rule, including source and destination prefixes and ports, action, and optional description

### AccessList

We'll need to define a few fields for our model.
A field is a piece of data associated with an instance of a model and is defined using a class attribute.
Each model gets a numeric primary key field (`id`) automatically, so we don't need to worry about that, but we do need to define fields for the ACL's name, default action, and optional comments.

```python
class AccessList(NetBoxModel):
    name = models.CharField(
        max_length=100,
    )
    default_action = models.CharField(
        max_length=30,
    )
    comments = models.TextField(
        blank=True,
    )
```

By default, Django orders records by the primary key.
For access lists, ordering by name is usually more natural.
We can do that by adding a `Meta` class inside `AccessList`.

We will also set `verbose_name`, which controls how the model name is displayed in the UI.

```python
    class Meta:
        ordering = ('name',)
        verbose_name = 'Access List'
```

Finally, add a `__str__()` method so the model has a friendly string representation.

```python
    def __str__(self):
        return self.name
```

At this point, `models.py` should look like this:

```python
from django.contrib.postgres.fields import ArrayField
from django.db import models
from netbox.models import NetBoxModel


class AccessList(NetBoxModel):
    name = models.CharField(
        max_length=100,
    )
    default_action = models.CharField(
        max_length=30,
    )
    comments = models.TextField(
        blank=True,
    )

    class Meta:
        ordering = ('name',)
        verbose_name = 'Access List'

    def __str__(self):
        return self.name
```

### AccessListRule

Next, we will define a model for the individual rules that belong to an access list.

Each rule needs the following fields:

* Parent access list (a foreign key to an `AccessList` instance)
* Index (rule order)
* Protocol
* Source prefix
* Source ports
* Destination prefix
* Destination ports
* Action (permit, deny, reject)
* Description (optional)

Start by creating the model and adding a `ForeignKey` back to `AccessList`:

```python
class AccessListRule(NetBoxModel):
    access_list = models.ForeignKey(
        to='netbox_access_lists.AccessList',
        on_delete=models.CASCADE,
        related_name='rules',
    )
```

A quick breakdown of the arguments we passed:

* `to` identifies the related model. Using the string form `'<app_label>.<ModelName>'` avoids import order issues and matches how we reference models in other apps.
* `on_delete=models.CASCADE` means: if an access list is deleted, delete its rules as well.
* `related_name='rules'` creates a reverse relationship so you can access rules from an access list instance via `acl.rules.all()`.

Next, add an `index` field to store the rule's position within the access list.
We'll use `PositiveIntegerField` because only positive numbers are supported:

```python
    index = models.PositiveIntegerField()
```

Now add the protocol field.
This will store the name of a protocol such as TCP or UDP.
Notice that we're setting `blank=True` because it should not be required to specify a particular protocol when creating a rule.

```python
    protocol = models.CharField(
        max_length=30,
        blank=True,
    )
```

Next, define a source prefix.
We reference NetBox's [`Prefix` model](https://netboxlabs.com/docs/netbox/en/stable/models/ipam/prefix/) using the `'<app_label>.<ModelName>'` format, in this case `ipam.Prefix`.
We also mark it optional using `blank=True` and `null=True`.

```python
    source_prefix = models.ForeignKey(
        to='ipam.Prefix',
        on_delete=models.PROTECT,
        related_name='+',
        blank=True,
        null=True,
    )
```

:green_circle: **Tip:** `PROTECT` prevents deletion of the referenced object while it is still in use. This is often safer than allowing accidental deletes of shared objects like prefixes.

Notice `related_name='+'` above.
This tells Django not to create a reverse relationship from `Prefix` back to `AccessListRule`, which keeps the `Prefix` model namespace cleaner.

Now add a field for source ports.
We want to support more than one port per rule, so we will store a list of integers using [`ArrayField`](https://docs.djangoproject.com/en/stable/ref/contrib/postgres/fields/#arrayfield) to store a list of `PositiveIntegerField` values.

```python
    source_ports = ArrayField(
        base_field=models.PositiveIntegerField(),
        blank=True,
        null=True,
    )
```

Now add destination prefix and destination ports, which mirror the source fields:

```python
    destination_prefix = models.ForeignKey(
        to='ipam.Prefix',
        on_delete=models.PROTECT,
        related_name='+',
        blank=True,
        null=True,
    )
    destination_ports = ArrayField(
        base_field=models.PositiveIntegerField(),
        blank=True,
        null=True,
    )
```

Finally, add the rule action (required) and an optional description:

```python
    action = models.CharField(
        max_length=30,
    )
    description = models.CharField(
        max_length=500,
        blank=True,
    )
```

Add a `Meta` class to:

* order rules by access list and index
* enforce uniqueness so two rules in the same access list cannot share the same index
* set a friendly model name for the UI

```python
    class Meta:
        ordering = ('access_list', 'index')
        unique_together = ('access_list', 'index')
        verbose_name = 'Access List Rule'
```

Finally, we'll add a `__str__()` method to display the parent access list and index number when rendering an `AccessListRule` instance as a string:

```python
    def __str__(self):
        return f'{self.access_list}: Rule {self.index}'
```

## Define Field Choices

Some of our fields should only allow specific values.
For example, we want an `action` to be one of:

* Permit
* Deny
* Reject

NetBox provides [`ChoiceSet`](https://netboxlabs.com/docs/netbox/en/stable/plugins/development/models/#choice-sets) to define a reusable list of valid values (with optional colors for the UI).

Create a new file named `choices.py` next to `models.py`:

```bash
touch choices.py
```

In `choices.py`, import `ChoiceSet`:

```python
from utilities.choices import ChoiceSet
```

Now define `ActionChoices`:

```python
class ActionChoices(ChoiceSet):
    PERMIT = 'permit'
    DENY = 'deny'
    REJECT = 'reject'

    CHOICES = [
        (PERMIT, 'Permit', 'green'),
        (DENY, 'Deny', 'red'),
        (REJECT, 'Reject (Reset)', 'orange'),
    ]
```

The `CHOICES` attribute must be an iterable of two- or three-value tuples, each of which defines the following:

* the database value (e.g. `permit` or `deny`)
* the UI label (e.g. Permit, Deny, Reject)
* an optional UI color (see [available colors](https://netboxlabs.com/docs/netbox/en/stable/configuration/data-validation/#field_choices))

Now create `ProtocolChoices` as well:

```python
class ProtocolChoices(ChoiceSet):
    key = 'AccessListRule.protocol'

    TCP = 'tcp'
    UDP = 'udp'
    ICMP = 'icmp'

    CHOICES = [
        (TCP, 'TCP', 'blue'),
        (UDP, 'UDP', 'orange'),
        (ICMP, 'ICMP', 'purple'),
    ]
```

:blue_square: **Note:** We set `key` so an administrator can replace or extend these choices via NetBox's [`FIELD_CHOICES`](https://netboxlabs.com/docs/netbox/en/stable/configuration/data-validation/#field_choices) setting.

### Apply Choices in the Models

Back in `models.py`, import the choice sets:

```python
from .choices import ActionChoices, ProtocolChoices
```

Then update these fields to use the appropriate choices:

```python
    # AccessList
    default_action = models.CharField(
        max_length=30,
        choices=ActionChoices,
    )

    # AccessListRule
    protocol = models.CharField(
        max_length=30,
        choices=ProtocolChoices,
        blank=True,
    )
    # ...
    action = models.CharField(
        max_length=30,
        choices=ActionChoices,
    )
```

### Add Choice Color Methods

NetBox can display colors for choice values in tables and templates.
To support that, we add a method per choice field that returns the selected color.
This works similar to Django's `get_FOO_display()` methods, but returns a color (defined on the field's `ChoiceSet`) rather than a label.

Add this method to `AccessList`:

```python
class AccessList(NetBoxModel):
    # ...
    def get_default_action_color(self):
        return ActionChoices.colors.get(self.default_action)
```

Add these methods to `AccessListRule`:

```python
class AccessListRule(NetBoxModel):
    # ...
    def get_protocol_color(self):
        return ProtocolChoices.colors.get(self.protocol)

    def get_action_color(self):
        return ActionChoices.colors.get(self.action)
```

At this point, your `models.py` file should look like this:

```python
from django.contrib.postgres.fields import ArrayField
from django.db import models
from netbox.models import NetBoxModel

from .choices import ActionChoices, ProtocolChoices


class AccessList(NetBoxModel):
    name = models.CharField(
        max_length=100,
    )
    default_action = models.CharField(
        max_length=30,
        choices=ActionChoices,
    )
    comments = models.TextField(
        blank=True,
    )

    class Meta:
        ordering = ('name',)
        verbose_name = 'Access List'

    def __str__(self):
        return self.name

    def get_default_action_color(self):
        return ActionChoices.colors.get(self.default_action)


class AccessListRule(NetBoxModel):
    access_list = models.ForeignKey(
        to='netbox_access_lists.AccessList',
        on_delete=models.CASCADE,
        related_name='rules',
    )
    index = models.PositiveIntegerField()
    protocol = models.CharField(
        max_length=30,
        choices=ProtocolChoices,
        blank=True,
    )
    source_prefix = models.ForeignKey(
        to='ipam.Prefix',
        on_delete=models.PROTECT,
        related_name='+',
        blank=True,
        null=True,
    )
    source_ports = ArrayField(
        base_field=models.PositiveIntegerField(),
        blank=True,
        null=True,
    )
    destination_prefix = models.ForeignKey(
        to='ipam.Prefix',
        on_delete=models.PROTECT,
        related_name='+',
        blank=True,
        null=True,
    )
    destination_ports = ArrayField(
        base_field=models.PositiveIntegerField(),
        blank=True,
        null=True,
    )
    action = models.CharField(
        max_length=30,
        choices=ActionChoices,
    )
    description = models.CharField(
        max_length=500,
        blank=True,
    )

    class Meta:
        ordering = ('access_list', 'index')
        unique_together = ('access_list', 'index')
        verbose_name = 'Access List Rule'

    def __str__(self):
        return f'{self.access_list}: Rule {self.index}'

    def get_protocol_color(self):
        return ProtocolChoices.colors.get(self.protocol)

    def get_action_color(self):
        return ActionChoices.colors.get(self.action)
```

## Create Schema Migrations

Now that we have our models defined, we need to generate a schema for the PostgreSQL database.
While it's possible to create the tables and constraints by hand, it's _much_ easier to employ Django's [migrations feature](https://docs.djangoproject.com/en/stable/topics/migrations/).
This will inspect our model classes and generate the necessary migration files automatically.

This is a two-step process:

1. Generate migration files with `makemigrations`
2. Apply them to the database with `migrate`

:warning: **Warning:** Before continuing, confirm that `DEVELOPER=True` is set in NetBox's `configuration.py`. NetBox uses this setting to guard against accidental migration creation in non development environments.

### Generate Migration Files

Change into the NetBox installation root so you can run `manage.py` (for example `/opt/netbox`).

First, run `makemigrations` in dry run mode so you can review what will be generated:

```bash
(venv) $ python netbox/manage.py makemigrations netbox_access_lists --dry-run
Migrations for 'netbox_access_lists':
  ~/netbox-plugin-demo/netbox_access_lists/migrations/0001_initial.py
    + Create model AccessList
    + Create model AccessListRule
```

We should see a plan to create our plugin's first migration file, `0001_initial.py`, with the two models we defined in `models.py`.
(If you encounter an error at this point, or don't see the output above, **stop here** and review your work.)
If everything looks good, run it again without `--dry-run` to actually create the migration file:

```bash
(venv) $ python netbox/manage.py makemigrations netbox_access_lists
Migrations for 'netbox_access_lists':
  ~/netbox-plugin-demo/netbox_access_lists/migrations/0001_initial.py
    + Create model AccessList
    + Create model AccessListRule
```

Back in your plugin workspace, you should now see a `migrations` directory with `__init__.py` and `0001_initial.py`:

```bash
$ tree
.
├── netbox_access_lists
│   ├── choices.py
│   ├── __init__.py
│   ├── migrations
│   │   ├── 0001_initial.py
│   │   └── __init__.py
│   └── models.py
├── pyproject.toml
└── README.md
```

### Apply Migrations

Now apply the migrations from the NetBox root directory:

```bash
(venv) $ python netbox/manage.py migrate netbox_access_lists
Operations to perform:
  Apply all migrations: netbox_access_lists
Running migrations:
  Applying netbox_access_lists.0001_initial... OK
```

If you're curious, you can inspect the newly created database tables, using the `dbshell` management command to enter a PostgreSQL shell:

```bash
(venv) $ python netbox/manage.py dbshell
psql (16.11 (Ubuntu 16.11-0ubuntu0.24.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

netbox=> \d netbox_access_lists_accesslist
                             Table "public.netbox_access_lists_accesslist"
      Column       |           Type           | Collation | Nullable |             Default
-------------------+--------------------------+-----------+----------+----------------------------------
 id                | bigint                   |           | not null | generated by default as identity
 created           | timestamp with time zone |           |          |
 last_updated      | timestamp with time zone |           |          |
 custom_field_data | jsonb                    |           | not null |
 name              | character varying(100)   |           | not null |
 default_action    | character varying(30)    |           | not null |
 comments          | text                     |           | not null |
Indexes:
    "netbox_access_lists_accesslist_pkey" PRIMARY KEY, btree (id)
Referenced by:
    TABLE "netbox_access_lists_accesslistrule" CONSTRAINT "netbox_access_lists__access_list_id_6c1b0317_fk_netbox_ac" FOREIGN KEY (access_list_id) REFERENCES netbox_access_lists_accesslist(id) DEFERRABLE INITIALLY DEFERRED
```

Type `\q` to exit `dbshell`.

## Create Some Objects

Now that we have our models installed, let's try creating some objects.
First, enter the NetBox shell.
This is an interactive Python command line interface that allows us to interact directly with NetBox objects and other resources.

Start the NetBox interactive shell:

```bash
(venv) $ python netbox/manage.py nbshell
### NetBox interactive shell
### Python v3.12.3 | Django v5.2.10 | NetBox Community v4.5.2
### Plugins: netbox_access_lists v0.1
### lsapps() & lsmodels() will show available models. Use help(<model>) for more info.
>>>
```

Create an access list, validate it with `full_clean()`, then save it:

```python
>>> from netbox_access_lists.models import AccessList, AccessListRule
>>> from netbox_access_lists.choices import ActionChoices, ProtocolChoices
>>> acl = AccessList(name='MyACL1', default_action=ActionChoices.DENY)
>>> acl.full_clean()
>>> acl.save()
>>> acl
<AccessList: MyACL1>
```

Next, create a couple of prefixes to reference in rules:

```python
>>> from ipam.models import Prefix
>>> prefix1 = Prefix(prefix='192.168.1.0/24')
>>> prefix1.full_clean()
>>> prefix1.save()
>>> prefix2 = Prefix(prefix='192.168.2.0/24')
>>> prefix2.full_clean()
>>> prefix2.save()
```

:blue_square: **Note:** If `full_clean()` complains about missing required fields (for example `status`), set the missing values and try again. Defaults can vary depending on your NetBox version and configuration.

Now create two rules:

```python
>>> rule1 = AccessListRule(
...     access_list=acl,
...     index=10,
...     protocol=ProtocolChoices.TCP,
...     destination_prefix=prefix1,
...     destination_ports=[80, 443],
...     action=ActionChoices.PERMIT,
...     description='Web traffic'
... )
>>> rule1.full_clean()
>>> rule1.save()
>>> rule2 = AccessListRule(
...     access_list=acl,
...     index=20,
...     protocol=ProtocolChoices.UDP,
...     destination_prefix=prefix2,
...     destination_ports=[53],
...     action=ActionChoices.PERMIT,
...     description='DNS'
... )
>>> rule2.full_clean()
>>> rule2.save()
```

Verify the rules are associated with the access list.
We can use the `all()` manager to retrieve all rules belonging to a particular access list:

```python
>>> acl.rules.all()
<RestrictedQuerySet [<AccessListRule: MyACL1: Rule 10>, <AccessListRule: MyACL1: Rule 20>]>
```

Exit the shell:

```python
>>> exit()
```

Excellent. We can now create access lists and rules in the database.
In the next steps, we will expose this functionality in the NetBox user interface.

<div align="center">

:arrow_left: [Step 1: Plugin Configuration](/tutorial/step01-plugin-configuration.md) | [Step 3: Tables](/tutorial/step03-tables.md) :arrow_right:

</div>

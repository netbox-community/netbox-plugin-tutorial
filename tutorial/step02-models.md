# Step 2: Models

In this step, we're going to define some Django models to hold our plugin's data. A model is a Python class that represents a table in the underlying PostgreSQL database; each instance of a model equates to a row in the table. We use models instead of raw SQL because interacting with Python objects is much more convenient and flexible.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step01-initial-setup`.

## Create the Models

First, `cd` into the `netbox_access_lists` directory and create a file named `models.py`. This is where our model classes will be defined.

```bash
$ cd netbox_access_lists
$ edit models.py
```

At the top of the file, import Django's `models` library and NetBox's `NetBoxModel` class. The latter will serve as the base class for our plugin's models. We'll also import the PostgreSQL `ArrayField`; more on this in a bit.

```python
from django.contrib.postgres.fields import ArrayField
from django.db import models
from netbox.models import NetBoxModel
```

We'll create two models:

* `AccessList`: This will represent an access list, with a name and one or more rules assigned to it.
* `AccessListRule`: This will be an individual rule with source/destination IP addresses and port numbers, etc. assigned to an access list.

### AccessList

We'll need to define a few fields for our model. Each model gets a numeric primary key field (`id`) automatically, so we don't need to worry about that, but we do need to define fields for the ACL's name, default action, and optional comments.

```python
class AccessList(NetBoxModel):
    name = models.CharField(
        max_length=100
    )
    default_action = models.CharField(
        max_length=30
    )
    comments = models.TextField(
        blank=True
    )
```

By default, model instances are ordered by their primary keys, but it would make more sense to order access lists by name. We can do that by creating a `Meta` subclass and defining an `ordering` variable. (Be sure to create the `Meta` class *inside* `AccessList`, not under it.)

```python
    class Meta:
        ordering = ('name',)
```

Finally, we'll add a `__str__()` method to control how an instance is rendered in a string. We'll have this return the value of the instance's `name` field. (Again, be sure to create this *inside* `AccessList`.)

```python
    def __str__(self):
        return self.name
```

### AccessListRule

Our second model will hold the individual rules assigned to each access list. This model will be a bit more complex. We'll need to define fields for:

* Parent access list (pointing to and `AccessList` instance)
* Index (the rule's order in the list)
* Protocol
* Source prefix
* Source port(s)
* Destination prefix
* Destination port(s)
* Action (permit, allow, or reject)
* Description (optional)

Let's start by defining a `ForeignKey` field pointing to the `AccessList` model.

```python
class AccessListRule(NetBoxModel):
    access_list = models.ForeignKey(
        to=AccessList,
        on_delete=models.CASCADE,
        related_name='rules'
    )
```

We're passing three keyword arguments to the field:

* `to` references the related model class (this can alternatively be the _name_ of the class)
* `on_delete` tells Django what action to take if the related object is deleted. `CASCADE` will automatically delete any rules assigned to it as well.
* `related_name` defines the attribute of the reverse relationship being added to the related class. The rule assigned to an `AccessList` instance can be referenced as `accesslist.rules.all()`.

Next we'll add an `index` field to store the rule's number (position) within the access list. We'll use `PositiveIntegerField` because only positive numbers are supported.

```python
    index = models.PositiveIntegerField()
```

The protocol field is next. This will store the name of a protocol such as `'tcp'` or `'udp'`. Notice that we're setting `blank=True` because it should not be required to specify a particular protocol when creating a rule.

```python
    protocol = models.CharField(
        max_length=30,
        blank=True
    )
```

:green_circle: **Tip:** Why didn't we set `null=True` like we did for the previous optional fields? Because this is a `CharField`, it's recommended to store empty values as empty strings rather than `null`. For other data types, like integers or booleans, `null` must be explicitly allowed at the database level for optional fields.

Next we need to define a source prefix. We're going to use a foreign key field to reference an instance of NetBox's `Prefix` model within its `ipam` app. Instead of importing the model class, we can just reference it by its name. And because we want this to be an _optional_ field, we'll also set `blank=True` and `null=True`.

```python
    source_prefix = models.ForeignKey(
        to='ipam.Prefix',
        on_delete=models.PROTECT,
        related_name='+',
        blank=True,
        null=True
    )
```

Notice above that we've defined `related_name='+'`. This tells Django not to create a reverse relationship from the `Prefix` model to the `AccessListRule` model, because it wouldn't be very useful.

We also need to add a field for the source port number(s). We could use an integer field for this, however that would limit us to defining a single source port per rule. Instead, we can add an `ArrayField` to store a list of `PositiveIntegerField` values. Like `source_prefix`, this will also be an optional field, so we add `blank=True` and `null=True` as well.

```python
    source_ports = ArrayField(
        base_field=models.PositiveIntegerField(),
        blank=True,
        null=True
    )
```

Let's go ahead an add destination prefix and port fields as well. These are essentially duplicates of our source fields.

```python
    destination_prefix = models.ForeignKey(
        to='ipam.Prefix',
        on_delete=models.PROTECT,
        related_name='+',
        blank=True,
        null=True
    )
    destination_ports = ArrayField(
        base_field=models.PositiveIntegerField(),
        blank=True,
        null=True
    )
```

Finally, we'll add fields for the rule's action and description. The action is required but a description is not.

```python
    action = models.CharField(
        max_length=30
    )
    description = models.CharField(
        max_length=500,
        blank=True
    )
```

With our fields out of the way, this model will also need a `Meta` class to define database ordering and to ensure that every rule has a unique index number within its parent access list.

```python
    class Meta:
        ordering = ('access_list', 'index')
        unique_together = ('access_list', 'index')
```

Finally, we'll add a `__str__()` method to display the parent access list and index number when rendering an instance as a string:

```python
    def __str__(self):
        return f'{self.access_list}: Rule {self.index}'
```

## Define Field Choices

Looking back at our models, we see a few fields that would benefit from having pre-defined choices from which a user can select when creating or modifying an instance. Specifically, we expect a rule's `action` field to only ever have one of three values:

* Permit
* Deny
* Reject

We can define a `ChoiceSet` to store these pre-defined values for the user, to avoid the hassle of manually typing the name of the desired value each time. Back at the top of `models.py`, import NetBox's `ChoiceSet` class:

```python
from utilities.choices import ChoiceSet
```

Then, below the import statements but above the model definitions, create a child class named `ActionChoices`:

```python
class ActionChoices(ChoiceSet):
    key = 'AccessListRule.action'

    CHOICES = (
        ('permit', 'Permit', 'green'),
        ('deny', 'Deny', 'red'),
        ('reject', 'Reject (Reset)', 'orange'),
    )
```

The `CHOICES` attribute is a tuple of tuples, each of which holds three values:

* The raw value to be stored in the database
* A human-friendly string for display
* A color for display in the UI (optional)

Additionally, we've added a `key` attribute: This will allow the NetBox administrator to replace or extend the plugin's default choices here with his or her own values.

Now, we can reference this as the set of valid choices on the `default_action` and `action` model fields by referencing it with the `choices` keyword argument.

```python
# AccessList
    default_action = models.CharField(
        max_length=30,
        choices=ActionChoices
    )

# AccessListRule
    action = models.CharField(
        max_length=30,
        choices=ActionChoices
    )
```

Let's create a set of choices for the rule model's `protocol` field as well. Add this below the `ActionChoices` class:

```python
class ProtocolChoices(ChoiceSet):

    CHOICES = (
        ('tcp', 'TCP', 'blue'),
        ('udp', 'UDP', 'orange'),
        ('icmp', 'ICMP', 'purple'),
    )
```

Then, add the `choices` keyword argument to the `protocol` field:

```python
# AccessListRule
    protocol = models.CharField(
        max_length=30,
        choices=ProtocolChoices,
        blank=True
    )
```

## Create Schema Migrations

Now that we have our models defined, we need to generate a schema for the PostgreSQL database. While it's possible to create the tables and constraints by hand, it's _much_ easier to employ Django's [migrations feature](https://docs.djangoproject.com/en/4.0/topics/migrations/), which will introspect our model classes and generate the necessary migration files automatically. This is a two-step process: First we generate the migration file with the `makemigrations` management command, then we run `migrate` to apply it to the live database.

:warning: **Warning:** Before continuing, check that you've set `DEVELOPER=True` in NetBox's `configuration.py` file. This is necessary to disable a safeguard intended to prevent people from creating new migrations mistakenly.

### Generate Migration Files

Change into the NetBox installation root to run `manage.py`. First, we'll run `makemigrations` with the `--dry-run` argument as a sanity-check: This will report what changes have been detected, but won't actually generate any migration files.

```bash
$ python netbox/manage.py makemigrations netbox_access_lists --dry-run
Migrations for 'netbox_access_lists':
  ~/netbox-plugin-demo/netbox_access_lists/migrations/0001_initial.py
    - Create model AccessList
    - Create model AccessListRule
```

We should see a plan to create our plugin's first migration file, `0001_initial.py`, with the two models we defined in `models.py`. (If you encounter an error at this point, or don't see the output above, **stop here** and review your work.) If everything looks good, proceed with creating the migration file (omitting the `--dry-run` argument):

```bash
$ python netbox/manage.py makemigrations netbox_access_lists
Migrations for 'netbox_access_lists':
  ~/netbox-plugin-demo/netbox_access_lists/migrations/0001_initial.py
    - Create model AccessList
    - Create model AccessListRule
```

Back in your plugin workspace, you should now see a `migrations` directory with two files: `__init__.py` and `0001_initial.py`.

```bash
$ tree
.
├── __init__.py
├── migrations
│   ├── 0001_initial.py
│   ├── __init__.py
...
```

### Apply Migrations

Finally, we can apply the migration file using the `migrate` management command:

```bash
$ python netbox/manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, circuits, contenttypes, dcim, django_rq, extras, ipam, netbox_access_lists, sessions, social_django, taggit, tenancy, users, virtualization, wireless
Running migrations:
  Applying netbox_access_lists.0001_initial... OK
```

If you're curious, you can inspect the newly created database tables, using the `dbshell` management command to enter a PostgreSQL shell:

```bash
$ python netbox/manage.py dbshell
psql (10.19 (Ubuntu 10.19-0ubuntu0.18.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

netbox=> \d netbox_access_lists_accesslist
                                          Table "public.netbox_access_lists_accesslist"
      Column       |           Type           | Collation | Nullable |                          Default                           
-------------------+--------------------------+-----------+----------+------------------------------------------------------------
 id                | bigint                   |           | not null | nextval('netbox_access_lists_accesslist_id_seq'::regclass)
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

## Create Some Objects

Now that we have our models installed, let's try creating some objects. First, enter the NetBox shell. This is an interactive Python command line which allows us to interact directly with NetBox objects and other resources.

```bash
$ ./manage.py nbshell
from netbox### NetBox interactive shell
### Python 3.8.12 | Django 4.0.3 | NetBox 3.2.0
### lsmodels() will show available models. Use help(<model>) for more info.
>>>
```

Let's create and save an access list:

```python
>>> from netbox_access_lists.models import *
>>> acl = AccessList(name='MyACL1', default_action='deny')
>>> acl
<AccessList: MyACL1>
>>> acl.save()
```

And a few rules to go with it:

```python
>>> AccessListRule(
...     access_list=acl,
...     index=10,
...     protocol='tcp',
...     destination_prefix=prefix1,
...     destination_ports=[80, 443],
...     action='permit',
...     description='Web traffic'
... ).save()
>>> AccessListRule(
...     access_list=acl,
...     index=20,
...     protocol='dns',
...     destination_prefix=prefix2,
...     destination_ports=[53],
...     action='permit',
...     description='DNS'
... ).save()
>>> acl.rules.all()
<RestrictedQuerySet [<AccessListRule: MyACL1: Rule 10>, <AccessListRule: MyACL1: Rule 20>]>
```

Excellent! We can now create access lists and rules in the database. The next few steps will work on expsoing this functionality in the NetBox user interface.

:arrow-right: [Step 3: Tables](/tutorial/step03-tables.md)


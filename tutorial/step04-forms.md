# Step 4: Forms

Form classes generate HTML form elements for the user interface, and process and validate user input. They are used in NetBox primarily to create, modify, and delete objects. We'll create a form class for each of our plugin's models.

:blue_square: **Note:** If you skipped the previous step, run `git checkout step03-tables`.

## Create the Forms

Begin by creating a file named `forms.py` in the `netbox_access_lists/` directory.

```bash
$ cd netbox_access_lists/
$ edit forms.py
```

At the top of the file, we'll import NetBox's `NetBoxModelForm` class, which will serve as the base class for our forms. We'll also import our plugin's models.

```python
from netbox.forms import NetBoxModelForm
from .models import AccessList, AccessListRule
```

### AccessListForm

Create a class named `AccessListForm`, subclassing `NetBoxModelForm`. Under this class, define a `Meta` subclass defining the form's `model` and `fields`. Notice that the `fields` list also includes `tags`: Tag assignment is handled by `NetBoxModel` automatically, so we didn't need to add it to our model in step two.

```python
class AccessListForm(NetBoxModelForm):

    class Meta:
        model = AccessList
        fields = ('name', 'default_action', 'comments', 'tags')
```

This alone is sufficient for our first model, but we can make one tweak: Instead of the default field that Django will generate for the `comments` model field, we can use NetBox's purpose-built `CommentField` class. (This handles some largely cosmetic details like setting a `help_text` and adjusting the field's layout.) To do this, simply import the `CommentField` class and override the form field:

```python
from utilities.forms.fields import CommentField
# ...
class AccessListForm(NetBoxModelForm):
    comments = CommentField()

    class Meta:
        model = AccessList
        fields = ('name', 'default_action', 'comments', 'tags')
```

### AccessListRuleForm

We'll create a form for `AccessListRule` following the same pattern.

```python
class AccessListRuleForm(NetBoxModelForm):

    class Meta:
        model = AccessListRule
        fields = (
            'access_list', 'index', 'description', 'source_prefix', 'source_ports', 'destination_prefix',
            'destination_ports', 'protocol', 'action', 'tags',
        )
```

By default, Django will create a "static" foreign key field for related objects. This renders as a dropdown list that's pre-populated with _all_ available objects. As you can imagine, in a NetBox instance with many thousands of objects this can get rather unwieldy.

To avoid this, NetBox provides the `DynamicModelChoiceField` class. This renders foreign key fields using a special dynamic widget backed by NetBox's REST API. This avoids the overhead imposed by the static field, and allows the user to conveniently search for the desired object.

:green_circle: **Tip:** The `DynamicModelMultipleChoiceField` class is also available for many-to-many fields, which support the assignment of multiple objects.

We'll use `DynamicModelChoiceField` for the three foreign key fields in our form: `access_list`, `source_prefix`, and `destination_prefix`. First, we must import the field class, as well as the models of the related objects. `AccessList` is already imported, so we just need to import `Prefix` from NetBox's `ipam` app. The beginning of `forms.py` should now look like this:

```python
from ipam.models import Prefix
from netbox.forms import NetBoxModelForm
from utilities.forms.fields import CommentField, DynamicModelChoiceField
from .models import AccessList, AccessListRule
```

Then, we override the three relevant fields on the form class, instantiating `DynamicModelChoiceField` with the appropriate `queryset` value for each. (Be sure to keep in place the `Meta` class we already defined.)

```python
class AccessListRuleForm(NetBoxModelForm):
    access_list = DynamicModelChoiceField(
        queryset=AccessList.objects.all()
    )
    source_prefix = DynamicModelChoiceField(
        queryset=Prefix.objects.all()
    )
    destination_prefix = DynamicModelChoiceField(
        queryset=Prefix.objects.all()
    )
```

With our models, tables, and forms all in place, next we'll create some views to bring everything together!

<div align="center">

:arrow_left: [Step 3: Tables](/tutorial/step03-tables.md) | [Step 5: Views](/tutorial/step05-views.md) :arrow_right:

</div>


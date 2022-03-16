# NetBox Plugin Development Tutorial

This guide seeks to demonstrate the process of developing a custom plugin for NetBox v3.2 or later. By following each of the prescribed steps, the reader will create from scratch a simple plugin for managing access lists in NetBox, utilizing all major components of the NetBox plugin framework.

A completed copy of the demo plugin created in this tutorial is available in the [`netbox-plugin-demo`](https://github.com/netbox-community/netbox-plugin-demo) repository for reference. For your convenience, the completed code corresponding to each step in the tutorial exists as a named branch in the demo repo. For example, if you want to start fresh on step 5, simply check out the `step04-forms` branch.

### Prerequisites

Before attempting to create a plugin, please assess your personal ability. Plugin authors should have reasonable proficiency in the following:

* Python programming
* The [Django](https://www.djangoproject.com/) framework
* REST API fundamentals (where applicable)
* Installing, configuring, and using NetBox

### Contents

* [Step 1: Initial Setup](/tutorial/step01-initial-setup.md) :arrow_left: Start here!
* [Step 2: Models](/tutorial/step02-models.md)
* [Step 3: Tables](/tutorial/step03-tables.md)
* [Step 4: Forms](/tutorial/step04-forms.md)
* [Step 5: Views](/tutorial/step05-views.md)
* [Step 6: Templates](/tutorial/step06-templates.md)
* [Step 7: Navigation](/tutorial/step07-navigation.md)
* [Step 8: Filter Sets](/tutorial/step08-filter-sets.md)
* [Step 9: REST API](/tutorial/step09-rest-api.md)
* [Step 10: GraphQL API](/tutorial/step10-graphql-api.md)

### Reference

* [NetBox Plugin Development Documentation](https://netbox.readthedocs.io/en/feature/plugins/development/)

### Getting Help

If you run into any snags working through the tutorial, please join us in the **#netbox** channel on the [NetDev Community Slack](https://netdev.chat/) for help.

### Feedback and Issues

If you happen to uncover an error or discrepancy in the tutorial, please be sure to [open an issue](https://github.com/netbox-community/netbox-plugin-tutorial/issues/new/choose) so that it can be documented and fixed.


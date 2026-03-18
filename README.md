# NetBox Plugin Development Tutorial

This guide seeks to demonstrate the process of developing a custom plugin for NetBox v4.5 or later.
By following each of the prescribed steps, the reader will create from scratch a simple plugin for managing access lists in NetBox, using all major components of the NetBox plugin framework.

A completed copy of the demo plugin created in this tutorial is available in the [`netbox-plugin-demo`](https://github.com/netbox-community/netbox-plugin-demo) repository for reference.
For your convenience, the completed code corresponding to each step in the tutorial exists as a named branch in the demo repo.
For example, if you want to start fresh on step 5, check out the `step04-forms` branch.

### Prerequisites

Before attempting to create a plugin, please assess your personal ability. Plugin authors should have reasonable proficiency in the following:

* Python programming
* The [Django](https://www.djangoproject.com/) framework
* REST API fundamentals (where applicable)
* Installing, configuring, and using NetBox

### Contents

* [Step 0: Initial Setup](/tutorial/step00-initial-setup.md) :arrow_left: Start here!
* [Step 1: Plugin Definition](/tutorial/step01-plugin-definition.md)
* [Step 2: Models](/tutorial/step02-models.md)
* [Step 3: Tables](/tutorial/step03-tables.md)
* [Step 4: Forms](/tutorial/step04-forms.md)
* [Step 5: Views](/tutorial/step05-views.md)
* [Step 6: URLs](/tutorial/step06-urls.md)
* [Step 7: Templates](/tutorial/step07-templates.md)
* [Step 8: Navigation](/tutorial/step08-navigation.md)
* [Step 9: Filter Sets](/tutorial/step09-filter-sets.md)
* [Step 10: REST API](/tutorial/step10-rest-api.md)
* [Step 11: GraphQL API](/tutorial/step11-graphql-api.md)
* [Step 12: Search](/tutorial/step12-search.md)
* [Step 13: Wrap Up](/tutorial/step13-wrap-up.md)

### Reference

* [NetBox Plugin Development Documentation](https://netboxlabs.com/docs/netbox/en/stable/plugins/development/)
* [NetBox Labs Certified Plugin Program](https://github.com/netbox-community/netbox/wiki/Plugin-Certification-Program)

### Getting Help

If you run into any snags working through the tutorial, please join us in the **#netbox** channel on the [NetDev Community Slack](https://netdev.chat/) for help.

### Feedback and Issues

If you happen to uncover an error or discrepancy in the tutorial, please be sure to [open an issue](https://github.com/netbox-community/netbox-plugin-tutorial/issues/new/choose) so that it can be documented and fixed.


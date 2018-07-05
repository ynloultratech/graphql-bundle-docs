It's very important keep your code and your dependencies up to date. 
Upgrade a framework to a newer version can sometimes be a headache. 
We will explain in this guide how the versions in this Bundle works and how upgrade.

>> GrapQLBundle use [semantic versioning](https://semver.org/) 
with some important aspects that must be taken into account when upgrading.

First of all, you need to know that a change in our library not only may represents a change in your source code, 
it can also lead to a major change in the consumers of your API. 

In order to keep BC with your source code and your API consumer this Bundle 

|Version|Description|
|---|---|
|PATCH (1.0.**X**)|Bug fixes and minor improvements. Does not affect your code or your API consumer in any way.|
|MINOR (1.**X**.0)|Release features, deprecations notices etc. **This version does not have any breaking change with your code; but may require some adjustments to keep BC with your API consumers. Must read [UPGRADE](../98_UPGRADE.md) guide.**|
|MAYOR (**X**.0.0)|Release features, remove deprecated features etc. **This version surely contains breaking changes with your code, and can affect your API consumers. Must read [UPGRADE](../98_UPGRADE.md) guide**|

# Upgrade to PATCH Version

Upgrade to a **PATCH** version is always safe. No additional changes are required in your code and does not affect
your API consumers at all.

- **Source Code:** Not Affected
- **API Schema:** Not Affected

>>> Is Recommended for production environments use a [constraint](https://getcomposer.org/doc/articles/versions.md#hyphenated-version-range-)
 like `~1.2.0` or `>=1.2.0 <1.3.0` in your `composer.json`. This ensures you have the latest bugs fixed, but does not add changes that **MINOR** versions may have that can affect your API consumers.

# Upgrade to MINOR Version

Before upgrading to a **MINOR** version, check which version is currently installed. 
Then follow [UPGRADE](../98_UPGRADE.md) steps to make all necessary adjustments, if any.

A **MINOR** version always has fully compatibility with your source code, but sometimes these versions
add new features and depreciate others. The update may change your API schema and affect directly your API consumers
because deprecated features are hidden by default.

- **Source Code:** Not Affected
- **API Schema:** May contains Breaking Changes and require activate some BC options.

For example, before `v1.2` all lists had an argument called `filters` with basic options to filter collections, in
`v1.2` a new option called `where` was added. This new option is similar to the old one but is not compatible because
has some advanced features and definitions types are different. For that reason the old argument `filters` has been marked
has deprecated and hidden by default. 

This update does not require a change in your source code, but introduce a breaking change in your API that affect your consumers
because all consumers using the deprecated feature `filters` are affected.

### Why deprecated features are hidden in the schema?

Because new users installing directly the latest version of GraphQLBundle does not need see deprecated features in their schema, 
like the above example, for this users the `filters` option never existed, will only see the new option `where` in their schema.

### How keep BC with API consumers?

Because all deprecated features of previous versions are hidden in new versions,
can introduces a breaking change with your API consumers. 
In order to keep BC with them and make the upgrade in your clients smoothly 
GraphQLBundle comes with special configuration to keep deprecated features
in your schema during some time.

Using the previous example the following configuration keep the deprecated `filters` in your schema in order to keep BC with your API consumers.

````yaml
graphql:
    # Backward compatibility layer to keep deprecated features during some time to upgrade API consumers progressively.
    bc:
        # Keep deprecated "filters" argument in collections
        filters: true
````

>> Activating an obsolete feature allows you to continue using it, but this feature is still deprecated and appear in your schema and documentation as deprecated.

Although an deprecated option can still be used by activating the backward compatibility layer, must inform your clients of the need to update their code to use the new option, in this case `where` instead of `filters`.

GraphQL does not have a versioning system like REST API's, [it does not need it](https://graphql.org/learn/best-practices/#versioning). But this does not mean that you should not have control of the changes and inform your clients during how much time a depreciated feature will be available. For example, keep a public CHANGELOG of your API with added and deprecated features, and the time required to update, a good example is how GitHub keep a [Changelog](https://developer.github.com/v4/changelog/) and [Breaking Changes](https://developer.github.com/v4/breaking_changes/) of their API.

# Upgrade to MAYOR Version

Upgrade to **MAYOR** version is similar to how update **MINOR** version, must need read the [UPGRADE](../98_UPGRADE.md) 
guide. But in this case it may be that your source code is affected. Deprecated interfaces,
classes and methods are removed in this version.

- **Source Code:** May contains Breaking Changes and require some changes in your code
- **API Schema:** May contains Breaking Changes and require activate some BC options.

Like **MINOR** versions always contains options to keep your API functional with
backward compatibility.



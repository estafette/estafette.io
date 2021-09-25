---
title: "Build an extension"
description: "To reuse complex logic in your build pipelines build your own extensions"
weight: 2
---

An _Estafette_ extension is in essence just a containerized application. What's special is that it uses custom properties injected as environment variables to parameterize it's behaviour.

In the example below all non-reserved _properties_ are automatically turned into a environment variable prefixed with `ESTAFETTE_EXTENSION_`.

```yaml
bake:
  image: extensions/docker:dev
  action: build
  repositories:
  - estafette
```

In this example the `action` property is set as `ESTAFETTE_EXTENSION_ACTION` on the stage container; the `repositories` property is set as `ESTAFETTE_EXTENSION_REPOSITORIES` and it's array items are joined into comma-separated value;

Once you start to increase the number of parameters it's more sensible to make use of environment variable `ESTAFETTE_EXTENSION_CUSTOM_PROPERTIES` into which all custom properties of the stage are json serialized.
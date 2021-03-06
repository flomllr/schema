---
id: plugin-nullabilityGuard
title: Nullability Guard Plugin
sidebar_label: Nullability Guard
---

This plugin helps us guard against non-null values crashing our queries in production. It does this by defining that every scalar value have a "fallback" value defined, so if we see a nullish value on an otherwise non-null field, we will fallback to this instead of crashing the query.

<blockquote class="warn">
<b>Note:</b>

The `nullabilityGuardPlugin` by default only guards when `process.env.NODE_ENV === 'production'`. This is intended so you see/catch these errors in development and do not use it except as a last resort. If you want to change this, set the `shouldGuard` config option.

</blockquote>

### Example Use:

```ts
import { nullabilityGuardPlugin } from "@nexus/schema";

const guardPlugin = nullabilityGuardPlugin({
  onNullGuarded(ctx, info) {
    // This could report to a service like Sentry, or log internally - up to you!
    console.error(
      `Error: Saw a null value for non-null field ${info.parentType.name}.${
        info.fieldName
      } ${root ? `(${root.id || root._id})` : ""}`
    );
  },
  // A map of `typeNames` to the values we want to replace with if a "null" value
  // is seen in a position it shouldn't be. These can also be provided as a config property
  // for the `objectType` / `enumType` definition, as seen below.
  fallbackValues: {
    Int: () => 0,
    String: () => "",
    ID: ({ info }) => `${info.parentType.name}:N/A`,
    Boolean: () => false,
    Float: () => 0,
  },
});
```

### Null Guard Algorithm

- If a field is nullable:

  - If the field is non-list, do not guard
  - If the field is a list, and none of the list members are nullable, do not guard

- If the field is non-nullable and the value is null:

  - If the field is a list:
    - If the value is nullish, return an empty list `[]`
    - If the list is non-empty, iterate and complete with a valid non-null fallback
  - If the value is a Union/Interface

    - Return with an object with the `__typename` of the first type which implements this contract

  - If the field is an object:
    - If the value is nullish
      - If there is a fallback defined on the object for that type, return with that
      - Else return with an empty object
    - Return the value and push forward to the next resolvers

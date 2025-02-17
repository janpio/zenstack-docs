---
description: Understanding access policies.
sidebar_position: 3
---

# Understanding Access Policies

Proper authorization is the key to a secure application. ZenStack allows you to define access policies directly inside your data model, so it's easier to keep the policies in sync when your data models evolve.

This document introduces the access policy enforcement behavior regarding different database operations. The general principle is to make the behavior a natural extension of how Prisma behaves today.

:::tip

Access policies are only effective when you call Prisma methods enhanced with [`enhance`](/docs/reference/runtime-api#enhance) or [`withPolicy`](/docs/reference/runtime-api#withpolicy).

:::

## General rules

:::info

Field-level access policies are in preview stage and its behavior may change in the future. Please let us know your feedback!

:::

Access policies are expressed with the `@@allow`/`@@deny` model attributes or `@allow`/`@deny` field attributes. The attributes take two parameters. The first is the operation: create/read/update/delete (only read/update for field-level policies). You can use a comma-separated string to pass multiple operations or use 'all' to abbreviate all operations. The second parameter is a boolean expression indicating if the rule should be activated.

```zmodel
attribute @@allow(_ operation: String, _ condition: Boolean)

attribute @allow(_ operation: String, _ condition: Boolean)

attribute @@deny(_ operation: String, _ condition: Boolean)

attribute @deny(_ operation: String, _ condition: Boolean)
```

`@@allow`/`@allow` opens up permissions while `@@deny`/`@deny` turns them off. You can use them multiple times and combine them in a model. Whether an operation is permitted is determined as follows:

For model-level policies:

1. If any `@@deny` rule evaluates to true, it's denied.
1. If any `@@allow` rule evaluates to true, it's allowed.
1. Otherwise, it's denied.

For field-level policies:

1. If any `@deny` rule evaluates to true, it's denied.
1. If there exists `@allow` rules and at least one of them evaluates to true, it's allowed.
1. If there exists `@allow` rules but none of them evaluates to true, it's denied.
1. Otherwise, it's allowed.

Please note the difference between model-level and field-level rules. Model-level access are by-default denied, while field-level access are by-default allowed. E.g., if you don't specify any "read" rule for a model, the model is not readable. However, if you don't specify any "read" rule for a field, the field is readable.


## Accessing user data

When using `enhance` to wrap a Prisma client for authorization, you pass in a context object containing the data about the current user (verified by authentication). This user
data can be accessed with the special `auth()` function in access policy rules. Note that `auth()` function's return value is typed as the `User` model in your schema, so only fields defined in the `User` model are accessible.

You can access its field to implement RBAC like:

```zmodel
model Post {
    // full access for admins
    // "role" field must be defined in the "User" model
    @@allow('all', auth().role == 'Admin')
}
```

, or you can use it to check the user's identity directly.

```zmodel
model Post {
    ...
    author User @relation(fields: [authorId], references: [id])
    @@allow('all', author == auth())
}
```

## Operations

### Create

#### Prisma methods

`create`, `createMany`, `upsert`

#### Behavior

"Create" operation is governed by the _**create**_ rules. Since an entity doesn't exist before it's created, the fields used in such rules implicitly refer to the creation result. [Field validation](/docs/reference/zmodel-language#field-validation) is also considered a part of _**create**_ rules.

When a create operation is rejected, a [`PrismaClientKnownRequestError`](https://www.prisma.io/docs/reference/api-reference/error-reference#prismaclientknownrequesterror) is thrown with code [`P2004`](https://www.prisma.io/docs/reference/api-reference/error-reference#p2004).

"Create" operations can contain nested creates, and if a nested-created model has _**create**_ rules, they're also enforced. The entire "create" happens in a transaction.

### Read

#### Prisma methods

`findUnique`, `findUniqueOrThrow`, `findFirst`, `findFirstOrThrow`, `findMany`

#### Behavior

"Read" operations are filtered by _**read**_ rules. For `findMany`, entities failing policy checks are silently dropped. For `findUnique` and `findFirst`, `null` is returned if the requested entity exists but fails policy checks. For `findUniqueOrThrow` and `findFirstOrThrow`, an error is thrown if the requested entity exists but fails policy checks.

```zmodel
model Foo {
    id String @id
    value Int
    @@allow('read', value > 0)
}
```

```ts
// given there's a single Foo { id: "1", value: 0 } in the database

db.foo.findUnique({ where: { id: '1' } }); // => null
db.foo.findUniqueOrThrow({ where: { id: '1' } }); // => throws
db.foo.findFirst(); // => null
db.foo.findFirstOrThrow(); // => throws
db.foo.findMany(); // => []
```

The _**read**_ rules also determine if the result of a mutation - create, update or delete can be read back. Therefore, even if a mutation succeeds (and is persisted), the call can still result in a [`PrismaClientKnownRequestError`](https://www.prisma.io/docs/reference/api-reference/error-reference#prismaclientknownrequesterror) because the entity being returned doesn't satisfy _**read**_ rules.

```zmodel
model Foo {
    id String @id
    value Int
    @@allow('read', value > 0)
}
```

```ts
// an entity is created in the database, but the call eventually throws because the result cannot be read back
const created = await db.foo.create({ data: { value: 0 } });
```

Field-level read rules don't affect the readability of model entities as a whole, however the annotated field is omitted from the result if it fails the checks.

```zmodel
model Foo {
    id String @id
    value Int @allow('read', value > 0)
}
```

```ts
// given there's a single Foo { id: "1", value: 0 } in the database
db.foo.findUnique({ where: { id: '1' } }); // => { id: '1' }
```

### Update

#### Prisma methods

`update`, `updateMany`, `upsert`

#### Behavior

"Update" operations are governed by the _**update**_ rules. An entity has a "pre-update" and "post-update" state. Fields used in _**update**_ rules implicitly refer to the "pre-update" state, and you can use the `future()` function to refer to the "post-update" state. [Field validation](/docs/reference/zmodel-language#field-validation) is also considered a part of _**update**_ rules.

```zmodel
model Post {
    id String @id
    title String @length(1, 100)
    author User @relation(fields: [authorId], references: [id])
    authorId String

    // "author" refers to "pre-update" and "future().author" refers to "post-update"
    @@allow('update', author == auth() && future().author == author)
}
```

For top-level or nested `updateMany`, access policies are used to "trim" the scope of the update (by merging with the "where" clause provided by the user). This can end up with fewer entities being updated than without policies. For unique update, either with a top-level `update` or a nested `update` to "to-one" relation, the update will be rejected if it fails policy checks. When an update operation is rejected, a [`PrismaClientKnownRequestError`](https://www.prisma.io/docs/reference/api-reference/error-reference#prismaclientknownrequesterror) is thrown with code [`P2004`](https://www.prisma.io/docs/reference/api-reference/error-reference#p2004).

```zmodel
model Foo {
    id String @id
    value Int

    @@allow('create,read', true)
    @@allow('update', value > 0)
}
```

```ts
// create a Foo { id: '1', value: 0 }
await db.foo.create({ data: { id: '1', value: 0 } });

// succeeds without updating anything
await db.foo.updateMany({ data: { value: 1 } }); // => { count: 0 }

// throws
await db.foo.update({ where: { id: '1' }, data: { value: 1 } });
```

Update operations can contain nested writes - creates, updates or deletes, and if a nested-written model has corresponding rules, they're also enforced. The entire update happens in a transaction.

```zmodel
model User {
    id String @id
    email String
    profile Profile?

    @@allow('all', true)
}

model Profile {
    id String @id
    user User @relation(fields: [userId], references: [id])
    userId String
    age Int

    @@allow('update', future().age > 0)
}
```

```ts
// throws because the nested update of "Profile" fails policy
// neither the "User" nor the "Profile" is updated
await db.user.update({
    where: { id: '1' },
    data: {
        email: 'abc@xyz.com',
        profile: {
            update: {
                age: 0,
            },
        },
    },
});
```

The handling of field-level update rules is the same as model-level ones. The only difference is that the rules are only activated when the annotated field is part of the update operation. In another word, when a field is set to be updated, its update rules are merged with model-level rules.

### Delete

#### Prisma methods

`delete`, `deleteMany`

#### Behavior

Delete operations are governed by the _**delete**_ rules. Since an entity doesn't exist after it's deleted, the fields used in such rules implicitly refer to the "pre-delete" state.

For top-level or nested `deleteMany`, access policies are used to "trim" the scope of delete (by merging with the "where" clause provided by the user). This can end up with fewer entities being deleted than without policies. For unique delete, either with top-level `delete` or nested `delete` to "to-one" relation, the deletion will be rejected if it fails policy checks.

### Aggregation

#### Prisma methods

`aggregate`, `groupBy`, `count`

#### Behavior

Aggregation operations are filtered by _**read**_ rules. Entities failing policy checks are silently excluded from the data set used for computing the aggregation result.

---
title: Apollo Federation subgraph specification
description: For implementing subgraphs in other languages
---

> This content is provided for developers adding federated subgraph support to a GraphQL server library, and for anyone curious about the inner workings of federation. It is _not_ required if you're already using a [subgraph-compatible library](./supported-subgraphs/) like Apollo Server.
>
> Servers that are partially or fully compatible with this specification are tracked in Apollo's [subgraph compatibility repository](https://github.com/apollographql/apollo-federation-subgraph-compatibility).

To make a GraphQL service subgraph-capable, it needs the following:

* Implementation of the federation schema specification
* Support for fetching service capabilities
* Implementation of stub type generation for references
* Implementation of request resolving for entities.

## Federation schema specification

Federated services will need to implement the following additions to the schema to allow the gateway to use the service for execution:

```graphql
scalar _Any
scalar FieldSet
scalar link__Import

# a union of all types that use the @key directive
union _Entity

enum link__Purpose {
  """
  `SECURITY` features provide metadata necessary to securely resolve fields.
  """
  SECURITY

  """
  `EXECUTION` features provide metadata necessary for operation execution.
  """
  EXECUTION
}

type _Service {
  sdl: String
}

extend type Query {
  _entities(representations: [_Any!]!): [_Entity]!
  _service: _Service!
}

directive @external on FIELD_DEFINITION
directive @requires(fields: FieldSet!) on FIELD_DEFINITION
directive @provides(fields: FieldSet!) on FIELD_DEFINITION
directive @key(fields: FieldSet!, resolvable: Boolean = true) repeatable on OBJECT | INTERFACE
directive @link(url: String, as: String, for: link__Purpose, import: [link__Import]) repeatable on SCHEMA
directive @shareable on OBJECT | FIELD_DEFINITION
directive @inaccessible on FIELD_DEFINITION | OBJECT | INTERFACE | UNION | ARGUMENT_DEFINITION | SCALAR | ENUM | ENUM_VALUE | INPUT_OBJECT | INPUT_FIELD_DEFINITION
directive @override(from: String!) on FIELD_DEFINITION

# this is an optional directive discussed below
directive @extends on OBJECT | INTERFACE
```

For more information on these additions, see the [glossary](#schema-modifications-glossary).

## Fetch service capabilities

Schema composition at the gateway requires having each service's schema, annotated with its federation configuration. This information is fetched from each service using `_service`, an enhanced introspection entry point added to the query root of each federated service.

> Note that the `_service` field is not exposed by the gateway; it is solely for internal use.

The `_service` resolver should return the `_Service` type which has a single field called `sdl`. This SDL (schema definition language) is a printed version of the service's schema **including** the annotations of federation directives. This SDL does **not** include the additions of the federation spec above. Given an input like this:

```graphql
extend type Query {
  me: User
}

type User @key(fields: "id") {
  id: ID!
}
```

The generated SDL should match that exactly with no additions. It is important to preserve the type extensions and directive locations and to omit the federation types.

Some libraries such as `graphql-java` don't have native support for type extensions in their printer. Apollo Federation supports using an `@extends` directive in place of `extend type` to annotate type references:

```graphql {1}
type User @key(fields: "id") @extends {
  id: ID! @external
  reviews: [Review]
}
```

## Create stub types

Individual federated services should be runnable without having the entire graph present. Fields marked with `@external` are declarations of fields that are defined in another service. All fields referred to in `@key`, `@requires`, and `@provides` directives need to have corresponding `@external` fields in the same service. This allows us to be explicit about dependencies on another service, and service composition will verify that an `@external` field matches up with the original field definition, which can catch mistakes or migration issues (when the original field changes its type for example). `@external` fields also give an individual service the type information it needs to validate and decode incoming representations (this is especially important for custom scalars), without requiring the composed graph schema to be available at runtime in each service.

A federated service should take the `@external` fields and types and create them locally so the service can run on its own.

## Resolve requests for entities

Execution of a federated graph requires being able to "enter" into a service at an entity type. To do this, federated services need to do two things:

* Make each entity in the schema part of the `_Entity` union
* Implement the `_entities` field on the query root

To implement the `_Entity` union, each type annotated with `@key` should be added to the `_Entity` union. If no types are annotated with the key directive, then the `_Entity` union and `Query._entities` field should be removed from the schema. For example, given the following partial schema:

```graphql
type Review @key(fields: "id") {
  id: ID!
  body: String
  author: User
  product: Product
}

extend type User @key(fields: "email") {
  email: String! @external
}

extend type Product @key(fields: "upc") {
  upc: String! @external
}
```

The `_Entity` union for that partial schema should be the following:

```graphql
union _Entity = Review | User | Product
```

The `_Entity` union is critical to support the `_entities` root field:

```graphql
_entities(representations: [_Any!]!): [_Entity]!
```

Queries across service boundaries will start off from the `_entities` root field. The resolver for this field receives a list of representations. A representation is a blob of data that is supposed to match the combined requirements of the fields requested on an entity.

For example, if we execute a query for the top product's `reviews`:

```graphql
query GetTopProductReviews {
  topProducts {
    reviews {
      body
    }
  }
}
```

The gateway will first fetch the `topProducts` from the Products service, asking for the `upc` of each product:

```graphql
query {
  topProducts {
    upc
  }
}
```

The reason it requests `upc` is because that field is specified as a requirement on `reviews`:

```graphql
extend type Product @key(fields: "upc") {
  upc: String @external
  reviews: [Review]
}
```

The gateway will then send a list of representations for the fetched products to the Reviews service:

```json
{
  "query": ...,
  "variables": {
    "_representations": [
      {
        "__typename": "Product",
        "upc": "B00005N5PF"
      },
      ...
    ]
  }
}
```

```graphql
query ($_representations: [_Any!]!) {
  _entities(representations: $_representations) {
    ... on Product {
      reviews {
        body
      }
    }
  }
}
```

GraphQL execution will then go over each representation in the list, use the `__typename` to match type conditions, build up a merged selection set, and execute it. Here, the inline fragment on `Product` will match and the `reviews` resolver will be called repeatedly with the representation for each product. Since `Product` is part of the `_Entity` union, it can be selected as a return of the `_entities` resolver.

To ensure the required fields are provided, and of the right type, the source object properties should coerce the input into the expected type for the property (by calling parseValue on the scalar type).

The real resolver will then be able to access the required properties from the (partial) object:

```graphql
{
  Product: {
    reviews(object) {
      return fetchReviewsForProductWithUPC(object.upc);
    }
  }
}
```

## Schema modifications glossary

### `type _Service`

A new object type called `_Service` must be created. This type must have an `sdl: String!` field which exposes the SDL of the service's schema

### `Query._service`

A new field must be added to the query root called `_service`. This field must return a non-nullable `_Service` type. The `_service` field on the query root must return SDL which includes all of the service's types (after any non-federation transforms), as well as federation directive annotations on the fields and types. The federation schema modifications (i.e. new types and directive definitions) *should not be* included in this SDL.

### `union _Entity`

A new union called `_Entity` must be created. This should be a union of all types that use the `@key` directive, including both types native to the schema and extended types.

### `scalar _Any`

A new scalar called `_Any` must be created. The `_Any` scalar is used to pass representations of entities from external services into the root `_entities` field for execution. Validation of the `_Any` scalar is done by matching the `__typename` and `@external` fields defined in the schema.

### `scalar FieldSet`

A new scalar called `FieldSet` is a custom scalar type that is used to represent a set of fields. Grammatically, a field set is a [selection set](http://spec.graphql.org/draft/#sec-Selection-Sets) minus the braces. This means it can represent a single field `"upc"`, multiple fields `"id countryCode"`, and even nested selection sets `"id organization { id }"`.

### `Query._entities`

A new field must be added to the query root called `_entities`. This field must return a non-nullable list of `_Entity` types and have a single argument with an argument name of `representations` and type `[_Any!]!` (non-nullable list of non-nullable `_Any` scalars). The `_entities` field on the query root must allow a list of `_Any` scalars which are "representations" of entities from external services. These representations should be validated with the following rules:

* Any representation without a `__typename: String` field is invalid.
* Representations must contain at least the fields defined in the fieldset of a `@key` directive on the base type.

### `@key`

```graphql
directive @key(fields: FieldSet!, resolvable: Boolean = true) repeatable on OBJECT | INTERFACE
```

The `@key` directive is used to indicate a combination of fields that can be used to uniquely identify and fetch an object or interface.

```graphql
type Product @key(fields: "upc") {
  upc: UPC!
  name: String
}
```

Multiple keys can be defined on a single object type:

```graphql
type Product @key(fields: "upc") @key(fields: "sku") {
  upc: UPC!
  sku: SKU!
  name: String
}
```

> Note: Repeated directives (in this case, `@key`, used multiple times) require support by the underlying GraphQL implementation.

### `@link`

```graphql
directive @link(
  url: String,
  as: String,
  for: link__Purpose,
  import: [link__Import]
) repeatable on SCHEMA
```

The `@link` directive links definitions within the document to external schemas.

External schemas are identified by their `url`, which optionally ends with a name and version with the following format: `{NAME}`/v`{MAJOR}`.`{MINOR}`

The presence of a `@link` directive makes a document a [core schema](https://specs.apollo.dev/#def-core-schema).

The `for` argument describes the purpose of a `@link`. Currently accepted values are `SECURITY` or `EXECUTION`. Core schema-aware servers such as Apollo Router and Gateway will refuse to operate on schemas that contain `@link`s to unsupported specs which are `for: SECURITY` or `for: EXECUTION`.

By default, `@link`ed definitions will be namespaced, i.e., `@federation__requires`. The `as` argument lets you pick the name for this namespace:
```graphql
extend schema @link(url: "https://specs.apollo.dev/federation/v2.0", as: "fed2")
type User {
  metrics: Metrics @external
  favorites: UserFavorites @fed2__requires(fields: "metrics")
}
```

The `import` argument enables you to import external definitions into your namespace, so they are not prefixed.

```graphql
    schema
      @link(url: "https://specs.apollo.dev/link/v1.0")
      @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key", "@requires", "@provides", "@external", { name: "@tag", as: "@mytag" }, "@extends", "@shareable", "@inaccessible", "@override"])
```

In the example above, we import various directives from `federation/v2.0` into our namespace. We also rename one of them, bringing in federation's `@tag` as `@mytag` to distinguish it from a different `@tag` directive already in the schema.

### `@provides`

```graphql
directive @provides(fields: FieldSet!) on FIELD_DEFINITION
```

The `@provides` directive is used to annotate the expected returned fieldset from a field on a base type that is guaranteed to be selectable by the gateway. Given the following example:

```graphql
type Review @key(fields: "id") {
  product: Product @provides(fields: "name")
}

extend type Product @key(fields: "upc") {
  upc: String @external
  name: String @external
}
```

When fetching `Review.product` from the Reviews service, it is possible to request the `name` with the expectation that the Reviews service can provide it when going from review to product. `Product.name` is an external field on an external type which is why the local type extension of `Product` and annotation of `name` is required.

### `@requires`

```graphql
directive @requires(fields: FieldSet!) on FIELD_DEFINITION
```

The `@requires` directive is used to annotate the required input fieldset from a base type for a resolver. It is used to develop a query plan where the required fields may not be needed by the client, but the service may need additional information from other services. For example:

```graphql
# extended from the Users service
extend type User @key(fields: "id") {
  id: ID! @external
  email: String @external
  reviews: [Review] @requires(fields: "email")
}
```

In this case, the Reviews service adds new capabilities to the `User` type by providing a list of `reviews` related to a user. In order to fetch these reviews, the Reviews service needs to know the `email` of the `User` from the Users service in order to look up the reviews. This means the `reviews` field / resolver *requires* the `email` field from the base `User` type.

### `@external`

```graphql
directive @external on FIELD_DEFINITION
```

The `@external` directive is used to mark a field as owned by another service. This allows service A to use fields from service B while also knowing at runtime the types of that field. For example:

```graphql
# extended from the Users service
extend type User @key(fields: "email") {
  email: String @external
  reviews: [Review]
}
```

This type extension in the Reviews service extends the `User` type from the Users service. It extends it for the purpose of adding a new field called `reviews`, which returns a list of `Review`s.

### `@shareable`

```graphql
directive @shareable on FIELD_DEFINITION | OBJECT
```

The `@shareable` directive is used to indicate that a field can be resolved by multiple subgraphs. Any subgraph that includes a shareable field can potentially resolve a query for that field.  To successfully compose, a field must have the same shareability mode (either shareable or non-shareable) across all subgraphs.

Any field using the [`@key` directive](#key) is automatically shareable. Adding the `@shareable` directive to an object is equivalent to marking each field on the object `@shareable`.

```graphql
type Product @key(fields: "upc") {
  upc: UPC!                         # shareable because upc is a key field
  name: String                      # non-shareable
  description: String @shareable    # shareable
}

type User @key(fields: "email") @shareable {
  email: String                    # shareable because User is marked shareable
  name: String                     # shareable because User is marked shareable
}
```

### `@override`

```graphql
directive @override(from: String!) on FIELD_DEFINITION
```

The `@override` directive is used to indicate that the current subgraph is taking responsibility for resolving the marked field _away_ from the subgraph specified in the `from` argument.

> Only one subgraph can `@override` any given field. If multiple subgraphs attempt to `@override` the same field, a composition error occurs.

The following example will result in all query plans made to resolve `User.name` to be directed to SubgraphB.

```graphql
# In SubgraphA
type User @key(fields: "id") {
  id: ID!
  name: String
}

# In SubgraphB
type User @key(fields: "id") {
  id: ID!
  name: String @override(from: "SubgraphA")
}
```

### `@inaccessible`
```graphql
directive @inaccessible(from: String!) on FIELD_DEFINITION | INTERFACE | OBJECT | UNION | ARGUMENT_DEFINITION | SCALAR | ENUM | ENUM_VALUE | INPUT_OBJECT | INPUT_FIELD_DEFINITION
```

The `@inaccessible` directive indicates that a location within the schema is inaccessible. Inaccessible elements are available to query at the subgraph level but are not available to query at the supergraph level (through the router or gateway). This directive enables you to preserve composition while adding the field to your remaining subgraphs. You can remove the @inaccessible directive when the rollout is complete and begin using the field.

```graphql
extend schema
    @link(url: "https://specs.apollo.dev/link/v1.0")
    @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@inaccessible"])


interface Product {
  id: ID!
  sku: String
  package: String
  createdBy: User
  hidden: String @inaccessible
}
```

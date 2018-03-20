---
id: guide
title: Quick Start Guide
---

This guide will help you to easily create GraphQL Schemas that covers all your needs. We'll start by giving you information on how to install Graphql Composer. After that we create Type Composer objects (GraphQLObjectTypes that are editable), edit the TypeComposers with custom fields, setup relations, and generate a schema that you can use.

At the end we'll add a relay support to the graphql.

We always try to make our documentation better, so if you have feedback on the Guide please let us know via [opening an Issue in our Github repository](https://github.com/nodkz/graphql-compose/issues/new)

## Getting started

* [Generate TypeComposer](generate-typecomposer.html)
* [Edit TypeComposer](edit-typecomposer.html)
* [Relations](relations.html)
* [Mutations](mutations.html)
* [Generate Schema](generating-schema.html)
* [Relay support](relay.html)

[GraphQL](http://graphql.org/) – is a query language for APIs. [graphql-js](https://github.com/graphql/graphql-js) is the reference implementation of GraphQL for nodejs which introduce GraphQL type system for describing schema _(definition over configuration)_ and executes queries on the server side. [express-graphql](https://github.com/graphql/express-graphql) is a HTTP server which gets request data, passes it to `graphql-js` and returned result passes to response.

**`graphql-compose`** – the _imperative tool_ which worked on top of `graphql-js`. It provides some methods for creating types and GraphQL Models (so I call types with a list of common resolvers) for further building of complex relations in your schema.

* provides methods for editing GraphQL output/input types (add/remove fields/args/interfaces)
* introduces `Resolver`s – the named graphql fieldConfigs, which can be used for finding, updating, removing records
* provides an easy way for creating relations between types via `Resolver`s
* provides converter from `OutputType` to `InputType`
* provides `projection` parser from AST
* provides `GraphQL schema language` for defining simple types
* adds additional types `Date`, `Json`

**`graphql-compose-[plugin]`** – is a _declarative generators/plugins_ that build on top of `graphql-compose`, which take some ORMs, schema definitions and creates GraphQL Models from them or modify existed GraphQL Types.

Type generator plugins:

* [graphql-compose-json](https://github.com/graphql-compose/graphql-compose-json) - generates GraphQL type from JSON (a good helper for wrapping REST APIs)
* [graphql-compose-mongoose](https://github.com/graphql-compose/graphql-compose-mongoose) - generates GraphQL types from mongoose (MongoDB models) with Resolvers.
* [graphql-compose-elasticsearch](https://github.com/graphql-compose/graphql-compose-elasticsearch) - generates GraphQL types from elastic mappings; ElasticSearch REST API proxy via GraphQL.
* [graphql-compose-aws](https://github.com/graphql-compose/graphql-compose-aws) - expose AWS Cloud API via GraphQL

Utility plugins:

* [graphql-compose-relay](https://github.com/graphql-compose/graphql-compose-relay) - reassemble GraphQL types with `Relay` specific things, like `Node` type and interface, `globalId`, `clientMutationId`.
* [graphql-compose-connection](https://github.com/graphql-compose/graphql-compose-connection) - generates `connection` Resolver from `findMany` and `count` Resolvers.
* [graphql-compose-dataloader](https://github.com/stoffern/graphql-compose-dataloader) - add DataLoader to graphql-composer resolvers.
* [graphql-compose-recompose](https://github.com/digithun/graphql-compose-recompose) - utility that wrap GraphQL compose to high order functional pattern [work in process].

## Example

city.js

```js
import { TypeComposer} from 'graphql-compose';
import { CountryTC } from './country';

export const CityTC = TypeComposer.create(`
  type City {
    code: String!
    name: String!
    population: Number
    countryCode: String
    tz: String
  }
`);

// Define some additional fields
CityTC.addFields({
  ucName: { // standard GraphQL like field definition
    type: GraphQLString,
    resolve: (source) => source.name.toUpperCase(),
  },
  currentLocalTime: { // extended GraphQL Compose field definition
    type: 'Date',
    resolve: (source) => moment().tz(source.tz).format(),
    projection: { tz: true }, // load `tz` from database, when requested only `localTime` field
  },
  counter: 'Int', // shortening for only type definition for field
  complex: `type ComplexType {
    subField1: String
    subField2: Float
    subField3: Boolean
    subField4: ID
    subField5: JSON
    subField6: Date
  }`,
  list0: {
    type: '[String]',
    description: 'Array of strings',
  },
  list1: '[String]',
  list2: ['String'],
  list3: [new GraphQLOutputType(...)],
  list4: [`type Complex2Type { f1: Float, f2: Int }`],
});

// Add resolver method
CityTC.addResolver({
  kind: 'query',
  name: 'findMany',
  args: {
    filter: `input CityFilterInput {
      code: String!
    }`,
    limit: {
      type: 'Int',
      defaultValue: 20,
    },
    skip: 'Int',
    // ... other args if needed
  },
  type: [CityTC], // array of cities
  resolve: async ({ args, context }) => {
    return context.someCityDB
      .findMany(args.filter)
      .limit(args.limit)
      .skip(args.skip);
  },
});

// Add relation between City and Country by `countryCode` field.
CityTC.addRelation( // GraphQL relation definition
  'country',
  () => ({
    resolver: CountryTC.getResolver('findOne'),
    args: {
      filter: source => ({ code: `${source.countryCode}` }),
    },
    projection: { countryCode: true },
  })
);

// Remove `tz` field from schema
CityTC.removeField('tz');

// Add description to field
CityTC.extendField('name', {
  description: 'City name',
});
```

schema.js

```js
import { GQC } from 'graphql-compose';
import { CityTC } from './city';
import { CountryTC } from './country';

GQC.rootQuery().addFields({
  cities: CityTC.getResolver('findMany'),
  country: CountryTC.getResolver('findOne'),
  currentTime: {
    type: 'Date',
    resolve: () => Date.now(),
  },
});

GQC.rootMutation().addFields({
  createCity: CityTC.getResolver('createOne'),
  updateCity: CityTC.getResolver('updateById'),
  ...adminAccess({
    removeCity: CityTC.getResolver('removeById'),
  }),
});

function adminAccess(resolvers) {
  Object.keys(resolvers).forEach(k => {
    resolvers[k] = resolvers[k].wrapResolve(next => rp => {
      // rp = resolveParams = { source, args, context, info }
      if (!rp.context.isAdmin) {
        throw new Error('You should be admin, to have access to this action.');
      }
      return next(rp);
    });
  });
  return resolvers;
}

export default GQC.buildSchema();
```

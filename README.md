# GraphQL-RxJs

fork of [graphql-js](https://github.com/graphql/graphql-js) which adds Observable support ([RxJs](http://reactivex.io/rxjs/)).

[![npm version](https://badge.fury.io/js/graphql-rxjs.svg)](http://badge.fury.io/js/graphql-rxjs)
[![Build Status](https://travis-ci.org/DxCx/graphql-rxjs.svg?branch=master)](https://travis-ci.org/DxCx/graphql-rxjs?branch=master)

## Intro

This package basiclly adds Observables support to a GraphQL resolver.
It is importing the original graphql-js package for all the code that is not patched,
and provide a reactive execution engine over it.

## Documentation

There isn't much to document, all of GraphQL documentation is relevant.
See the complete documentation at http://graphql.org/ and
http://graphql.org/graphql-js/.

## Versioning

I'll try to follow along with `graphql-js` versions,
so basiclly, each graphql-js version should have a working graphql-rxjs package with it.

## API

The library exports the following functions:

#### Observable support

```typescript
export function graphqlReactive(
  schema: GraphQLSchema,
  requestString: string,
  rootValue?: any,
  contextValue?: any,
  variableValues?: {[key: string]: any},
  operationName?: string
): Observable<ExecutionResult>;

export function executeReactive(
  schema: GraphQLSchema,
  document: DocumentNode,
  rootValue?: any,
  contextValue?: any,
  variableValues?: {[key: string]: any},
  operationName?: string
): Observable<ExecutionResult>;
```

The signature is equal to GraphQL original implementation (graphql + execute),
except it returns an observable instead of a promise.
The observable will stream immutable results.

#### Reactive Directives
```typescript
export function addReactiveDirectivesToSchema(
  schema: GraphQLSchema,
): void;
```

Calling this function on your existing `GraphQLSchema` object will
enable reactive directives suppot for the schema.
More information about reactive directives can be found below.

## Getting Started:

1. The Data source

  Let's start off by explaining our Observable, or Data-source that we are going to stream.
  ```typescript
  const clockSource = Observable.interval(1000).map(() => new Date()).publishReplay(1).refCount();
  ```

  We can see that it is an Observable that emits `i+1` every second:
  ```typescript
  let source = Observable.interval(1000);
  ```

  Next we are going to map the value to a the current timestep:
  ```typescript
  source = source.map(() => new Date());
  ```

  Finally, We are going to convert the observable into an hot observable, or multicast.

  This means that there is a ref count on the observable,
  only once the first subscriber subscribe, the provider function will be triggered,
  and from then on, each subscriber will get the last value (without triggering the provider function)
  and then, both subscribers will get `next` values.

  Once a subscriber unsubscribe, the refCount will decrease,
  until the counter reachs zero, and only then the unsubscribe function will be triggered.
  ```typescript
  const clockSource = source.publishReplay(1).refCount();
  ```

 This approach will let us manage resources much more efficiently.

2. The Scheme

  Next, let's look at our scheme

  ```graphql
  # Root Subscription
  type Query {
    someInt: Int
  }

  type Subscription {
    clock: String
  }
  ```

  `Query` type exposes `someInt`, just because it cannot be empty, let's ignore that.
  
  `Subscription` type expoes `clock` which is basiclly periodic timestamp strings.

3. Wrapping them togather

  So, here is a basic code to demonstrate the concept:
  ```typescript
  import { Observable } from 'rxjs';
  import { makeExecutableSchema } from 'graphql-tools';
  import { graphqlReactive } from 'graphql-rxjs';
  
  const clockSource = Observable.interval(1000).map(() => new Date()).publishReplay(1).refCount();

  const typeDefs = `
  # Root Subscription
  type Query {
	someInt: Int
  }

  type Subscription {
    clock: String
  }
  `;

  const resolvers = {
      Subscription: {
          clock(root, args, ctx) {
                return ctx.clockSource;
          },
      },
  };

  // Compose togather resolver and typeDefs.
  const scheme = makeExecutableSchema({typeDefs: typeDefs, resolvers: resolvers});

  // subscribe the clock
  const query = `
    subscription {
  	clock
    }
  `

  // Calling the reactive version of graphql
  graphqlReactive(scheme, query, null, { clockSource })
  .subscribe(console.log.bind(console), console.error.bind(console));
  ```

  The following code will emit in console:
  ```json
  {"data":{"clock":"Fri Feb 02 2017 20:28:01 GMT+0200 (IST)"}}
  {"data":{"clock":"Fri Feb 02 2017 20:28:02 GMT+0200 (IST)"}}
  {"data":{"clock":"Fri Feb 02 2017 20:28:03 GMT+0200 (IST)"}}
  ...
  ```

## Reactive Directives
This library also implements reactive directives, those are supported at the moment:

1. `GraphQLDeferDirective` (`@defer`)
  - This directive does not require any arguments.
  - This directive instructs the executor to not resolve this field immedaitly,
    but instead returning the response without the deferred field,
    once the field is deferred, it will emit a corrected result.
  - executor will ignore this directive if resolver for this value is not async.
  - can be applied on:
    - specific field
    - spread fragment
    - named fragment
  - Example: **(TODO)**
2. `GraphQLLiveDirective` (`@live`)
  - This directive does not require any arguments.
  - This directive instructs the executor that the value should be monitored live
    which means that once updated, it will emit the updated respose.
  - executor will ignore this directive if field is not resolved
    with an observable (or at least have a parent observable).
  - can be applied on:
    - specific field
    - spread fragment
    - named fragment
    - fragment definition - (Live Fragment)
  - Example: **(TODO)**

## Typescript support

Just install `@types/graphql` for initial GraphQL support,
then the package will automatically add typings for the new functions it provides.

## Issues

If you found an issue or have an idea, you are welcome to open a new ticket in [Issues Page](https://github.com/DxCx/graphql-rxjs/issues)

## Support

Using this approach, feels much more intuative then the other approaches to stream results so far.
Because of that I tried to push it into upstream `graphql-js` but got rejected,
if you want to support the project you can follow/thumbs up the following:

1. [Issue on graphql-js](https://github.com/graphql/graphql-js/issues/501)
2. [PR on graphql-js](https://github.com/graphql/graphql-js/pull/502)
3. [Draft design for apollo subscriptions](https://github.com/apollographql/graphql-subscriptions/pull/30)

### Contributing

All pull requests are welcome,
if you think something needs to be done, just open an issue or a PR :)

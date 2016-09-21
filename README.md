GraphQL HTTP Server Middleware
==============================

[![Build Status](https://travis-ci.org/graphql/express-graphql.svg?branch=master)](https://travis-ci.org/graphql/express-graphql)
[![Coverage Status](https://coveralls.io/repos/graphql/express-graphql/badge.svg?branch=master&service=github)](https://coveralls.io/github/graphql/express-graphql?branch=master)

Create a GraphQL HTTP server with any HTTP web framework that supports connect styled middleware, including [Connect](https://github.com/senchalabs/connect) itself and [Express](http://expressjs.com).

## Installation

```sh
npm install --save express-graphql
```

Then mount `express-graphql` as a route handler:

```js
const express = require('express');
const graphqlHTTP = require('express-graphql');

const app = express();

app.use('/graphql', graphqlHTTP({
  schema: MyGraphQLSchema,
  graphiql: true
}));

app.listen(4000);
```

## Options

The `graphqlHTTP` function accepts the following options:

  * **`schema`**: A `GraphQLSchema` instance from [`GraphQL.js`][].
    A `schema` *must* be provided.

  * **`graphiql`**: If `true`, presents [GraphiQL][] when the GraphQL endpoint is
    loaded in a browser. We recommend that you set
    `graphiql` to `true` when your app is in development, because it's
    quite useful. You may or may not want it in production.

  * **`rootValue`**: A value to pass as the `rootValue` to the `graphql()`
    function from [`GraphQL.js`][].

  * **`context`**: A value to pass as the `context` to the `graphql()`
    function from [`GraphQL.js`][]. If `context` is not provided, the
    `request` object is passed as the context.

  * **`pretty`**: If `true`, any JSON response will be pretty-printed.

  * **`formatError`**: An optional function which will be used to format any
    errors produced by fulfilling a GraphQL operation. If no function is
    provided, GraphQL's default spec-compliant [`formatError`][] function will be used.

  * **`validationRules`**: Optional additional validation rules queries must
    satisfy in addition to those defined by the GraphQL spec.

  * **`loadPersistedDocument`**: A function that takes an input id and returns a
    valid Document. If provided, this will allow your GraphQL endpoint to execute
    a document specified via `documentID`.

  * **`persistValidatedDocument`**: A function that takes a validated Document and
    returns an id that can be used to load it later. This is used in conjunction
    with `loadPersistedDocument`. Providing this function will enable persisting
    documents via the `document` parameter at the `/persist` subpath. It is
    recommended that this option only be enabled in development deployments.

## HTTP Usage

Once installed at a path, `express-graphql` will accept requests with
the parameters:

  * **`query`**: A string GraphQL document to be executed.

  * **`variables`**: The runtime values to use for any GraphQL query variables
    as a JSON object.

  * **`operationName`**: If the provided `query` contains multiple named
    operations, this specifies which operation should be executed. If not
    provided, a 400 error will be returned if the `query` contains multiple
    named operations.

  * **`raw`**: If the `graphiql` option is enabled and the `raw` parameter is
    provided raw JSON will always be returned instead of GraphiQL even when
    loaded from a browser.

GraphQL will first look for each parameter in the URL's query-string:

```
/graphql?query=query+getUser($id:ID){user(id:$id){name}}&variables={"id":"4"}
```

If not found in the query-string, it will look in the POST request body.

If a previous middleware has already parsed the POST body, the `request.body`
value will be used. Use [`multer`][] or a similar middleware to add support
for `multipart/form-data` content, which may be useful for GraphQL mutations
involving uploading files. See an [example using multer](https://github.com/graphql/express-graphql/blob/master/src/__tests__/http-test.js#L650).

If the POST body has not yet been parsed, express-graphql will interpret it
depending on the provided *Content-Type* header.

  * **`application/json`**: the POST body will be parsed as a JSON
    object of parameters.

  * **`application/x-www-form-urlencoded`**: this POST body will be
    parsed as a url-encoded string of key-value pairs.

  * **`application/graphql`**: The POST body will be parsed as GraphQL
    query string, which provides the `query` parameter.


## Combining with Other Express Middleware

By default, the express request is passed as the GraphQL `context`.
Since most express middleware operates by adding extra data to the
request object, this means you can use most express middleware just by inserting it before `graphqlHTTP` is mounted. This covers scenarios such as authenticating the user, handling file uploads, or mounting GraphQL on a dynamic endpoint.

This example uses [`express-session`][] to provide GraphQL with the currently logged-in session.

```js
const session = require('express-session');
const graphqlHTTP = require('express-graphql');

const app = express();

app.use(session({ secret: 'keyboard cat', cookie: { maxAge: 60000 }}));

app.use('/graphql', graphqlHTTP({
  schema: MySessionAwareGraphQLSchema,
  graphiql: true
}));
```

Then in your type definitions, you can access the request via the third "context" argument in your `resolve` function:

```js
new GraphQLObjectType({
  name: 'MyType',
  fields: {
    myField: {
      type: GraphQLString,
      resolve(parentValue, args, request) {
        // use `request.session` here
      }
    }
  }
});
```

## Persisted Documents

Putting control of the query into clients' hands is one of the primary benefits
of GraphQL. Though when this is deployed to production naively, it is possible to
end up uploading and revalidating the same document text across all clients of a
particular deployment. It can even be problematic within a single client, sending
the exact same query repeatedly. Uploading and validating a large GraphQL document
can be very costly, especially over a poor network connection from a mobile device
with something large and complex such as Facebook's News Feed GraphQL query.

One solution to this problem that was developed early in the use of GraphQL at
Facebook was persisted documents. At build time, we take the GraphQL document and
persist it on the server getting back an ID. Then at runtime, we only upload the
ID of the document and the variables. When the server receives this request, it
loads the query and executes it, knowing it was already validated.

## Debugging Tips

During development, it's useful to get more information from errors, such as
stack traces. Providing a function to `formatError` enables this:

```js
formatError: error => ({
  message: error.message,
  locations: error.locations,
  stack: error.stack
})
```

[`GraphQL.js`]: https://github.com/graphql/graphql-js
[`formatError`]: https://github.com/graphql/graphql-js/blob/master/src/error/formatError.js
[GraphiQL]: https://github.com/graphql/graphiql
[`multer`]: https://github.com/expressjs/multer
[`express-session`]: https://github.com/expressjs/session

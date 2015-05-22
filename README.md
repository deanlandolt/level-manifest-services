# level-manifest-services

Extended level-manifest format for defining methods as services

This lib defines a JSON-serializable format extending the one defined by `level-manifest` with some additional keys for describing services built on top of leveldb. It also provides some utilities for helping to surface `sublevels` defined with a compatible manifest over HTTP.

JSON-Schema is used to descibe the data model of records within a subdomain, and methods can be annotated with additional data to define their behavior when exposed over HTTP.


## Method Type Definitions

The types of the `arguments` and `return` for custom `async` methods exposed on a sublevel can be elaborated upon with schema definitions:

```js
{
  methods: {
    myCustomMethod: {
      type: 'async',
      // json-schema tuple typing to define arguments tuple for async functions
      // TODO: variadic calls, `...rest`
      arguments: [
        { type: 'string' },
        { type: 'number' }
      ],
      // schema defining type of value in success position of invoked callback
      return: {
        // implied `type: 'object'`
        properties: {
          foo: { type: 'string' }
        }
        // TODO: variadic returns, `spread...`, symmetrical with `...rest`
      },
      // possible errors can be documented as well (as an array with implied "oneOf" semantics)
      errors: [
        {
          type: 'error'
          name: 'NotFoundError',
          code: 404,
        },
        {
          type: 'error'
          name: 'FooError',
          code: -11
        },
        {
          type: 'string' // you can also emit non-errors (but, like, don't)
        }
      ]
    }
  }
}
```

### JSON-RPC

The above should be more than enough detail to allow clients to discover how to consume this method when exposed over a JSON-RPC endpoint. In fact, any method can be exposed via a JSON-RPC endpoint with no additional information other than a `level-manifest` type, but this is not terribly useful for consumers.


## JSON-REST

When exposing an arbitrary manifest-defined method as an endpoint in a REST-JSON API we need a little more information -- specifically, the endpoint path must be defined explicitly. The expected HTTP method in a custom method will default to `POST` (this makes the fewest assumptions about the underlying request), but this can be overriden. If overriding `GET` or `DELETE`, ensure the underlying path will not conflict with keys in the database -- method paths will be tested first, so this will shadow access to your underlying keys.

The mapping of typical REST conventions to the standard levelup methods (and some optional extension methods that offer additional primitives) is fairly straight-forward:

```js
// create or update a value by key
`PUT /{sublevels...}/{key}`

// request body with `content-type: 'application/json'`
// value as `JSON.parse` on `req.body` string
subdb.put(key, value, res)

// create a value with an auto-generated id
`POST /{sublevels...}/`

// only works if `create` exists on underlying sublevel db
subdb.create(key, value, res)

// alterantively we could defer to a key generation function
subdb.createKey(value, function (err, key) {
  if (err) return cb(err)
  subdb.put(key, value, cb)
})

// delete value by key
`DELETE /{sublevels...}/{key}`

// straightforward mapping to db's `del` method when keypath is provided
subdb.del(key, res)

// get record by key
`GET /{sublevels...}/{key}`

// value record returned as a stringified JSON object in object body
subdb.get(key, streamToString(JSON.stringifyObject(res)))

// no key path component yields a read stream
`GET /{sublevels...}/{?query}`

// query component is expected to be url-query-encoded `ltgt` query object
subdb.createReadStream(query, JSONStream.stringify(res))

// get a live stream for all changes or changes within a specific range
`SUBSCRIBE /{sublevels...}/{?query}`

// we need a base primitive for live streams in order to support this
subdb.createLiveStream(query)

// delete records within a key range
`DELETE /{sublevels...}/{?query}`

// ideally we can rely on as few base primitives as possible
util.del(subdb.createKeyStream(query))
```

Or we could just use `levelweb`, extending it to support things like `SUBSCRIBE`. But we still have to have a way to expose a method as a custom endpoint.

## Endpoint matching

The above defines the bae fallback behavior when exposing a sublevel as a JSON-REST API. Custom methods can be annotated with an `endpoint` key that defines a match configuration for requests. If a request matches this configuration it will parsed and passed to the matched method:

```
{
  methods: {
    myCustomMethod: {
      endpoint: {
        method: 'PATCH'
      }
      // ...
    }
  }
}
```

### Custom endpoint match tests

The functionality in the default match implementation is relatively lacking -- defining declarative patterns over HTTP requests is nontrivial. As a more powerful alternative, the match `test` can be provided directly, either as a function or a string which `eval`s to the match `test` function implementation:

```js
myCustomMethod: {
  endpoint: {
    // NOTE: there's no need to case-normalize `req.methods` and friends as
    // they'll be normalized before `test` is invoked
    test: function (req) { return req.method === 'PATCH' }

    // or as a string:
    test: 'function (req) { return req.method === 'PATCH' }'
  }
}
```

The endpoint test function will be provided a candidate `req` object and should return truthy if the method should be used.

Browserified modules available over HTTP may be referenced using JSON-Schema referencing:

```js
myCustomMethod: {
  endpoint: {
    // if module is a function:
    test: { $ref: '/path/to/endpoint/test/module' }
    // or if module is an object, use JSON-Pointer to point to function
    test: { $ref: '/path/to/my/endpoint/test/module#/my/fn/key' }
  }
}
```

## Endpoint req/res transforms

A request transform can be provided as a function or `eval`-able string value that should create an arguments tuple suitable for an underlying method given the provided HTTP `req` object. It will be invoked in the context of the source HTTP request, and will be passed the bound method as an argument. A `res` function may also be provided that will be invoked as the callback of an async function. It will be invoked within the context of the `res` stream instance.

TODO: more to work through

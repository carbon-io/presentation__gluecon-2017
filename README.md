# Gluecon, 2017.05.24, Boulder CO

## (0) Hello

* I'm Will Shulman, CEO and Co-founder of mLab.
* Database-as-a-Service for MongoDB that runs on AWS, Azure, and Google Cloud.
* We host more than half a million MongoDB deployments.
* Our infrastructure is large and complex, and is made up of dozens of types of microservices.
* We built Carbon.io to allow us to quickly build high-quality microservices.

## (1) What is Carbon.io?

* A Node.js framework for building commandline programs, microservices, and APIs.
* It is a framework, built on a set of core libraries.
* It is opinionated *(but also friendly)*.

## (2) Design goals

* Reduce boilerplate code and make it clear how to do the basics (e.g how to structure your application, define endpoints, validate parameters, etc...).
* Seamless integration with MongoDB (although one can use other dbs as well).
* Make control flow a non-issue (a.k.a. avoid callback hell).
* Easy authentication and access control.
* Easy to communicate with other microservices (and therefore easier to build distributed systems).
* Easy to document.
* Easy to test.

## (3) The Basics

### (3.1) Package structure

The Carbon.io package structure is that of a typical Node.js application.

```
<root>
  |
  |- package.json
  |- bin/ (optional)
  |- lib/
  |   |- MyService.js
  |   |...
  |- test/
  |   |- MyServiceTest.js
  |   |...
  |- docs/
```

### (3.2) Services, Endpoints, and Operations

In carbon.io the top-level application is called a *Service*.

* A Service is a tree of Endpoints
* Endpoints are a set of Operations (GET, PUT, POST, DELETE, etc...)


```
Service
  |
  |-- Endpoint_0
  |       | -- get
  |       | -- put
  |       | -- post
  |       | -- delete
  |       |-- Endpoint_0.1
  |       |       |-- get
  |       |       |-- post
  |       |-- Endpoint_0.2
  |       |       |-- get
  |-- Endpoint_1
  | ...
  |-- Endpoint_N
```

* Let's take a look at a real example: [Hello world example][1]

### (3.3) Defining parameters and responses

Operations can be decorated with structure that allows the system to automatically handle certain aspects of managing
inputs and outputs, and makes the API self-describing.

* Operations can formally define parameters (*query*, *header*, *path*, and *body* parameters).
* Operations can formally define responses by HTTP status code.
* Parameter and response definitions can define JSON Schemas that are automatically enforced.
* This is useful for:
  * Automatic validation of input parameters and response output
  * Generating API documentation
* Let's take a look: [Hello world (parameter parsing) example][2]

### (3.4) Working with MongoDB

While one can use Carbon.io with any database technology, Carbon.io makes it particularly easy to work with MongoDB.

#### (3.4.1) Connection management

Services centrally manage the MongoDB connection lifecycle for all MongoDB connections needed by the Service,
and ensure connection pools are managed properly.

Single database connection:

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    port: 8888,
    dbUri: "mongodb://localhost:27017/mydb",
    endpoints: {
      messages: o({
        _type: carbon.carbond.Endpoint,
        get: function(req) {
          return this.getService().db.getCollection('messages').find().toArray()
        }
      })
    }
  })
})
```

Multiple database connections:

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    port: 8888,
    dbUris: {
      main: "mongodb://localhost:27017/mydb",
      reporting: "mongodb://localhost:27017/reporting"
    }
    endpoints: {
      messages: o({
        _type: carbon.carbond.Endpoint,
        get: function(req) {
          return this.getService().dbs['main'].getCollection('messages').find().toArray()
        }
      }),
      dashboards: o({
        _type: carbon.carbond.Endpoint,
        get: function(req) {
          return this.getService().dbs['reporting'].getCollection('dashboards').find().toArray()
        }
      })
    }
  })
})
```

#### (3.4.2) Leafnode

* Part of Carbon.io
* Wrapper around standard Node.js MongoDB driver
* Supports both asynchronous and synchronous calling styles

#### (3.4.3) Putting it all together

Let's take a look at a real example that uses MongoDB and that illustrates a few other advanced features:

* [Hello world (mongodb)][3]


### (3.5) Working with other services

Carbon.io makes it very easy to write services that talk to other services.

* Let's  take a look: [Hello world (chaining)][4]

## (4) A step back: understanding Carbon's core operators (```o```, ```_o```, and ```__```)

The Carbon.io framework is built on top of a set of core libraries that, together, provide Carbon.io with
much of its power.

You will often see Carbon.io modules follow this general pattern:

```node
var carbon = require('carbon-io')
var __     = carbon.fibers.__(module)
var _o     = carbon.bond._o(module)
var o      = carbon.atom.o(module).main  // The .main variant is used for top-level modules that define 'main' entry points

__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    .
    .
    .
  })
})
```

The following sections will explain the purpose of:
* ```o```: the universal object factory.
* ```_o```: the universal name resolver.
* ```__```: light-weight Node.js "threads" called *Fibers*.

### (4.1) Atom (the ```o``` operator)

Atom is the universal object factory, and used to instantiate objects and to create *components*. Components are simply objects bound in the Node.js module namespace via ```module.exports```.

The easiest way to think about Atom is that it allows you create object literals that also define a *class binding*.

**Example**

Instead of defining a rectangle like this:
```node
{
  point1: { x: 1, y: 2 }
  point2: { x: 5, y: 7 }
}
```

You can define it like this:
```node
o({
  _type: Rectangle,
  point1: { x: 1, y: 2 }
  point2: { x: 5, y: 7 }
})
```

or even this:
```node
o({
  _type: Rectangle,
  point1: o({ _type: Point, x: 1, y: 2 })
  point2: o({ _type: Point, x: 5, y: 7 })
})
```

where ```Rectangle``` and ```Point``` are JavaScript classes with methods you can then invoke on these objects. For example:

```node
var r = o({
  _type: Rectangle,
  point1: o({ _type: Point, x: 1, y: 2 })
  point2: o({ _type: Point, x: 5, y: 7 })
})

r.scaleByFactor(10)
```

#### (4.1.1) The object lifecycle and ```_init```

Object creation via the ```o``` operator follows this sequence:

1. The ```_type``` field is evaluated. If it is a function it is then considered a constructor and a new instance of that Class is created by calling the constructor with no arguments. If it is an object, that object is used as the new object’s prototype. If no ```_type``` is supplied the default value of ```Object``` is used for ```_type```.
1. All field definitions in the object passed to the ```o``` operator are applied to the newly created object.
1. If the object has an _init method (either directly or via its Class), it is called.
1. The newly created object is returned

```node
o({
  dbUri: "mongodb://localhost:27017/mydb",
  db: undefined,
  _init: function() {
    try {
      this.db = carbon.leafnode.connect(this.dbUri)
    } catch (e) {
      throw new Error(`Error connecting to db: ${e.message}`)
    }
  }
})
```

#### (4.1.2) Dynamic properties

JavaScript allows for dynamic properties (via getters and setters) to be defined on objects via ```Object.defineProperty```. Atom has a shorthand for this via the ```$property``` keyword:

```node
o({
  now: {
    $property: {
      get: function() {
        return new Date()
      }
    }
  }
})
```

#### (4.1.3) Using Atom to write commandline programs using ```_main``` and ```cmdargs```

```node
var carbon = require('carbon-io')
var __     = carbon.fibers.__(module)
var _o     = carbon.bond._o(module)
var o      = carbon.atom.o(module).main  // Note the .main here since this is the main application

__(function() {
  module.exports = o({
    verbose: false,
    cmdargs: {
      sides: {
      abbr: "s",
      help: "The number of sides each die should have.",
      required: false,
      default: 6
    },
    num: {
      position: 0,
      flag: false,
      help: "The number of dice to roll.",
      required: false,
      default: 1
    },
    verbose: {
      abbr: "v",
      flag: true,
      help: "Log verbose output.",
      required: false,
      property: true // Will result in this.verbose having the value passed at the cmdline
    },
  },

  _main: function(options) {
    if (this.verbose) {
      console.log("Here is the input")
      console.log(args)
      console.log("Ok... rolling.......")
    }

    var numDice = args.num
    var numSides = args.sides
    var result = []
    for (var i = 0; i < numDice; i++) {
      result.push(Math.floor(Math.random() * numSides + 1)) // Random integer between 1 and numSides
    }

    console.log(result)
  }
})
```

* Note how we use ```o.main```.

[Let's see it in action][5]

#### (4.1.4) ```o.main```

* Special variant of the ```o``` operator.
* Calls ```_main``` iff ```require.main === module``` (i.e. the module is called as the main module).
* This allows us to create modules that can act both as applications *and* libraries (subtle point -- pretty cool).
   * We will see an example of this with the test framework

#### (4.1.5) Advantages of using Atom:

* Provide for a form of light weight dependency injection (a.k.a. DI).
* Provides for a generic form of application configuration.
* Declarative style ("what not how") via  "datastructure DSLs".

### (4.2) Bond (the ```_o``` operator)

Bond is the universal name resolver for carbon.io.

#### (4.2.1) Resolving environment variables
```node
_o('env:PORT')
```

#### (4.2.2) Resolving HTTP URLs
```node
_o('http:localhost:8888')
```

and using them to perform HTTP requests
```node
_o('http:localhost:8888').get().body
```

```node
_o('http:localhost:8888').getEndpoint('hello').post({ msg: "Hello world!" })
```

#### (4.2.3) Connecting to MongoDB via a MongoDB URI
```node
var mydb = _o('mongodb://localhost:27017/mydb')
mydb.getCollection("users").insert({email: "joe@acme.com"))
```

#### (4.2.4) Resolving other Node.js modules (like ```require```)
```node
_o('./HelloEndpoint')
```

#### In previous examples:
* [For environment variables][6]
* [For organizing modules][7]
* [For connecting to other services][8]

### (4.3) Fibers (the ```__``` operator)

Carbon.io uses [Node Fibers](https://github.com/laverdet/node-fibers) under the hood to manage the complexity
of Node.js concurrency. *Have you noticed any callbacks in the example code so far?*

Fibers allow you to write code that is *logically* synchronous. Consider the following code snippet:

```node
fs.readFile("foo.txt", function(err, data) {
  if (err) {
    console.log(err)
  } else {
    console.log(data)
  }
})
```

With Fibers you can do this:

```node
__(function() {
  try {
    var data = fs.readFile.sync("foo.txt") // control flow is yielded
    console.log(data)
  } catch (err) {
    console.log(err)
  }
})
```

* The ```readFile``` function blocks the fiber (yields) until data is returned.
* If an error occurs it is thown as an exception (with a useful stacktrace). This is huge.

#### (4.3.1) ```__```

The ```__``` operator is what spawns new fibers.

```node
__(function() {
  // Code that runs in fiber
  // Code in the context of the fiber may use .sync()
})
```

you can also optionally supply a callback:

```node
__(function() {
  // Code that runs in fiber
}, function(err, result) {
  // Code called when fiber returns
})
```

**Behavior**
* Code inside of a spwaned fiber runs asynchronous to the code that spawned the fiber
* If a callback is supplied, the return value from the function (or exception if thrown) is passed to the callback.
* From within the fiber ```.sync``` can be called to synchronously call functions (without *actually* blocking).

It should also be noted that you must use fibers in all top-level messages in the event loop. Examples:
* The main program
* Asynchronous functions (e.g. ```process.nextTick(function() { ... })```)
* In HTTP middleware to wrap the processing of an HTTP Request

#### (4.3.2) ```.sync```

The ```.sync``` method can be called to synchronously call an asynchronous function as long as that function takes the standard
errback function as its last argument.

* Call by omiting the last errback argument
* The value will be returned by function
* An exception will be thrown if there was an err

There are two forms of ```.sync```:

* ```OBJ.sync.METHOD(ARGS)```

Example
```node
fs.sync.readFile("foo.txt")
```

* ```OBJ.METHOD.sync(ARGS)```

Example
```node
fs.readFile.sync("foo.txt")
```

Best practice: The first form should be used if there is a receiver, and the second on plain functions.

#### (4.3.3) Creating synchronous wrappers

You can use ```.sync``` to make user-friendly synchronous wrappers:

```node
function readFile(path) {
  return fs.readfile.sync(path)
}
```

#### (4.3.4) ```__.ensure``` vs ```__.spawn```

There are two variants of the ```__``` operator, ```ensure``` and ```spawn```.

* ```__.ensure```: Only spawns a new fiber if not already executing within a fiber (**default**).
* ```__.spawn```: Always spawns a new fiber.

Using ```__.ensure``` is particularly useful as top-level wrappers for applications that you also want to be able
to use as components / libraries.

A great example of this are unit tests that you might want to both be part of a larger test suite as well as runnable
standalone.

```node
var carbon = require('carbon-io')
var __     = carbon.fibers.__(module) // default behavior is that of __.ensure
var o      = carbon.atom.o(module).main

__(function() {
  module.exports = o({
    _type: carbon.testtube.Test,
    doTest: function() {
      // test code here
    }
    tests: [
      _o('./SubTest1'),
      _o('./SubTest2'),
    ]
  })
})
```

### (4.3.5) Revisiting our examples

* [Hello world (mongodb)][9]
* [Hello world (chaining)][10]

### (4.3.6) Advantages and disadvantages

Advantages
* Very natural control flow model.
* Exceptions and stack traces.
* Makes Node.js more functional. *Wait what? Not passing functions as callbacks is more functional?*
  * A pillar of functional programming is that expressions evaluate to values.

Disadvantages
* Can't be used in the browser.
* While usually obvious, it is not generally clear when control flow will yield under the hood (*beware of shared mutable state*).

### (4.3.6) Future work (*no pun intended*)
* Better integration with Promises

## (5) Authentication and access control

* Authentication is about determining who the user is.
* Access control is about controlling what the user has permission to do.

### (5.1) Authentication

#### (5.1.1) The Authenticator class

To implement an Authenticator you create an instance of the ```Authenticator``` class:

```node
o({
  _type: carbon.carbond.security.Authenticator,
  authenticate: function(req) { // also supports cb if desired
    var user = figureOutWhoUserIs()
    return user
  }
})
```

When configured on a Service the authenticator will authenticate all requests and expose the
authenticated user via ```req.user``` in all endpoint operations.

#### (5.1.2) Built-in Authenticators

Carbon.io comes with several built-in Authenticators:

* ```HttpBasicAuthenticator```: Base class for implementing HTTP basic authentication.
  * ```MongoDBHttpBasicAuthenticator```: An ```HttpBasicAuthenticator``` backed by MongoDB.
* ```ApiKeyAuthenticator```: Base class for implementing API-key based authentication.
  * ```MongoDBApiKeyAuthenticator```: An ApiKeyAuthenticator backed by MongoDB.
* ```OauthAuthenticator```: *coming soon*

Example

```node
o({
  _type: carbon.carbond.Service,
  port: 8888,
  authenticator: o({
    _type: carbon.carbond.security.MongoDBApiKeyAuthenticator,
    apiKeyParameterName: "api_key",
    apiKeyLocation: "header",   // can be "header" or "query"
    userCollection: "users",    // mongodb collection where user objects are stored
    apiKeyUserField: "apiKey"   // field that contains user api keys
  }),
  endpoints: {
    hello: o({
      _type: carbon.carbond.Endpoint,
      get: function(req) {
        return { msg: `Hello ${req.user.email}!`}
      }
    })
  }
})
```

### (5.2) Access control

Endpoints can configure an Access Control List (ACL) to govern which users can perform each HTTP
operation (e.g. GET, PUT, POST, etc.) on that Endpoint.

```node
o({
  _type: carbon.carbond.Endpoint,
  acl: o({
    _type: carbon.carbond.security.EndpointAcl
      groupDefinitions: { // This ACL defines two groups, 'role' and 'title'.
        role: 'role'      // We define a group called 'role' based on the user property named 'role'.
        title: function(user) { return user.title }
      }
      entries: [
        {
          user: { role: "Admin" },
          permissions: {
            "*": true // "*" grants all permissions
          }
        },
        {
          user: { title: "CFO" },
          permissions: { // We could have used "*" here but are being explicit.
            get: true,
            post: true
          }
        },
        {
          user: "12345", // User with _id "12345"
          permissions: {
            get: false,
            post: true
          }
        },
        {
          user: "*" // All other users
          permissions: {
            get: true,
            post: false,
          }
        }
      ]
    }),
    get: function(req) {
      return { msg: "Hello World!" }
    },
    post: function(req) {
      return { msg: `Hello ${req.body}!` }
    }
  })
})
```

### (5.2) Putting it all together
* [Hello world AAC example][11]


## (6) Collections

Collections are an abstraction on top of ```Endpoint```s that provide a higher-level interface for implementing
access to a collection of resources.

Common use case:
```
GET /users         // Get all Users
POST /users        // Add a User
GET /users/123     // Get User with _id of 123
PUT /users/123     // Modify User with _id of 123
DELETE /users/123  // Remove User with _id of 123
```

### (6.1) The Collection interface

When implementing a ```Collection```, instead of implementing the low-level HTTP methods
(```get```, ```post```, ```delete```, etc...), you implement the following higher-level interface:

* ```insert(obj, reqCtx)```
* ```find(query, reqCtx)```
* ```update(query, update, reqCtx)```
* ```remove(query, reqCtx)```
* ```saveObject(obj, reqCtx)```
* ```findObject(id, reqCtx)```
* ```updateObject(id, update, reqCtx)```
* ```removeObject(id, reqCtx)```

Which results in the following tree of ```Endpoint```s and ```Operation```s:

* ```/<collection>/```
   * ```POST``` which maps to ```insert```
   * ```GET``` which maps to ```find```
   * ```PATCH``` which maps to ```update```
   * ```DELETE``` which maps to ```remove```
* ```/<collection>/:_id```
   * ```PUT``` which maps to ```saveObject```
   * ```GET``` which maps to ```findObject```
   * ```PATCH``` which maps to ```updateObject```
   * ```DELETE``` which maps to ```removeObject```

### (6.2) MongoDBCollection

The ```MongoDBCollection``` class is a ```Collection``` that is backed by a MongoDB database collection.

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    port: 8888,
    dbUri: 'mongodb://localhost:27017/mydb',
    endpoints: {
      messages: o({
        _type: carbon.carbond.mongodb.MongoDBCollection,
        collection: 'messages'
      })
    }
  })
})
```

Let's look at a more elaborate example:
* [Zipcode service][12]

### (6.3) Custom Collections

You can create custom collections that implement the ```Collection``` interface however you like:

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.Service,
    port: 8888,
    dbUri: "mongodb://localhost:27017/mydb",
    endpoints: {
      feedback: o({
        _type: carbon.carbond.collections.Collection,
        // POST /feedback
        insert: function(obj) {
          obj.ts = new Date()
          this.getService().db.getCollection("feedback").insert(obj)
        },
      })
    }
  })
})
```

Let's look at a more elaborate example:
* [Contact service][13]

## (7) Testing with Test-tube

Carbon.io comes with a testing library called Test-tube.

### (7.1) Basic test structure

All tests have the following structure:

```node
var carbon = require('carbon-io')
var __ = carbon.fibers.__(module)
var o = carbon.atom.o(module).main
var testtube = carbon.testtube
var assert = require('assert')

__(function() {
  module.exports = o({
    _type: testtube.Test,
    name: 'ExampleTest',
    description: 'A simple test example.',
    setup: function(context, done) {},    // optional
    teardown: function(context, done) {}, // optional
    tests: [],                            // sub-tests (optional)
    doTest: function(context, done) {     // optional as well - usually you implement this or have sub-tests
       // test implementation
    }
  })
})
```

Test implementations (as well as ```setup``` and ```teardown```) can be synchronous or asynchronous.

Synchronous
```node
__(function() {
  module.exports = o({
    _type: testtube.Test,
    name: 'ExampleTest',
    description: 'A simple test example.',
    doTest: function(context) {
       assert(1 == 1)
    }
  })
})
```

Asynchronous
```node
__(function() {
  module.exports = o({
    _type: testtube.Test,
    name: 'ExampleTest',
    description: 'A simple test example.',
    doTest: function(context, done) {
      someAsyncFunction(function(err, value) {
        if (err) {
          done(e)
        } else {
          try {
            assert(value == 0)
          } catch (e) {
            done(e)
          }
          done()
        }
      })
    })
  })
})
```

### (7.2) Test suites

Since all ```Test``` objects can have an array of child / sub-tests, Tests are trees. This makes
it easy to manage large test suites.

```node
var __ = require('@carbon-io/carbon-core').fibers.__(module)
var o = require('@carbon-io/carbon-core').atom.o(module).main // Since this can run as main
var _o = require('@carbon-io/carbon-core').bond._o(module)
var testtube = require('@carbon-io/carbon-core').testtube

__(function() {
  module.exports = o({
    _type: testtube.Test,
    name: "Math test suite",
    tests: [
      _o('./AdditionTests'),
      _o('./SubtractionTests'),
      _o('./MultiplicationTests'),
      _o('./DivisionTests')
    ],
  })
})
```

* Test trees can be arbitrarily deep.
* Any test node in the tree can be run individually.
* [Let's play with an example][14].

### (7.3) HttpTests

Test-tube makes it particularly easy to write HTTP-based tests.

```node
__(function() {
  module.exports = o({
    _type: testtube.HttpTest,
    name: "HttpTests",
    description: "Http tests",
    baseUrl: "http://localhost:8888",
    tests: [
      {
        reqSpec: {
          url: "/hello",
          method: 'GET'
        },
        resSpec: {
          statusCode: 200,
          body: "Hello world!"
        }
      },
    ]
  })
})
```

### (7.4) ServiceTests

The ```ServiceTest``` class is an extension of ```HttpTest``` that makes it easy to have the test start and stop your ```Service```
as part of the test process.

```node
__(function() {
  module.exports = o({
    _type: carbon.carbond.test.ServiceTest,
    name: "HttpTests",
    description: "Http tests",
    service: _o('./HelloService'),
    tests: [
      {
        reqSpec: {
          url: "/hello",
          method: 'GET'
        },
        resSpec: {
          statusCode: 200,
          body: "Hello world!"
        }
      },
    ]
  })
})
```

This test will:
1. Start ```HelloService```.
1. Run the HTTP tests defined in ```tests```.
1. Stop ```HelloService```.

### (7.4) Running tests

```shell
$ node test/HelloServiceTest
```

## (8) Generating API documentation for your Services

Each ```Service``` is capable of generating its own docs.

Flavors:
* Github Flavored Markdown
* Static HTML [using aglio](https://github.com/danielgtaylor/aglio)
* *We will be adding plug-in model*

```shell
$ node lib/HelloService.js gen-static-docs -h

Usage: /usr/local/bin/node HelloService.js gen-static-docs [options]

Options:
   -v VERBOSITY, --verbosity VERBOSITY   verbosity level (trace | debug | info | warn | error | fatal)
   --flavor FLAVOR                       choose your flavor (github-flavored-markdown | aglio)  [github-flavored-markdown]
   --out PATH                            path to write static docs to (directory for multiple pages (default: api-docs) and file for single page (default: README.md))
   -o OPTION, --option OPTION            set generator specific options (format is: option[:value](,option[:value])*, can be specified multiple times)
   --show-options                        show generator specific options

generate docs for the api
Environment variables:
  <none>

```

Example
```shell
$ node lib/HelloService.js gen-static-docs --flavor aglio --out api.html
```

* [Example output][15]

## (9) Should I use carbon.io in production?

While we at mLab do, we do not suggest using Carbon.io for production until the 1.0 release.

## (10) Questions?


## (11) Additional Resources

* https://github.com/carbon-io/presentation__gluecon-2017 (this talk)
* https://docs.carbon.io
* https://www.npmjs.com/package/carbon-io
* https://github.com/carbon-io/carbon-io
* https://github.com/carbon-io-examples
* will@mlab.com




[1]: https://github.com/carbon-io-examples/example__hello-world-service/tree/carbon-0.6
[2]: https://github.com/carbon-io-examples/example__hello-world-service-parameter-parsing/tree/carbon-0.6
[3]: https://github.com/carbon-io-examples/example__hello-world-service-mongodb/tree/carbon-0.6
[4]: https://github.com/carbon-io-examples/example__hello-world-service-chaining/tree/carbon-0.6
[5]: https://github.com/carbon-io-examples/example__simple-cmdline-app/tree/carbon-0.6
[6]: https://github.com/carbon-io-examples/example__hello-world-service-mongodb/tree/carbon-0.6/lib/HelloService.js#L51
[7]: https://github.com/carbon-io-examples/example__hello-world-service-mongodb/tree/carbon-0.6/lib/HelloService.js#L57-L58
[8]: https://github.com/carbon-io-examples/example__hello-world-service-chaining/tree/carbon-0.6/lib/PublicHelloService.js#L32
[9]: https://github.com/carbon-io-examples/example__hello-world-service-mongodb/tree/carbon-0.6/lib/HelloEndpoint.js#L50
[10]: https://github.com/carbon-io-examples/example__hello-world-service-chaining/tree/carbon-0.6/lib/PublicHelloService.js#L58
[11]: https://github.com/carbon-io-examples/example__hello-world-service-aac/tree/carbon-0.6
[12]: https://github.com/carbon-io-examples/example__zipcode-service/tree/carbon-0.6
[13]: https://github.com/carbon-io-examples/contacts-service-advanced/tree/carbon-0.6
[14]: https://github.com/carbon-io-examples/example__test-suites/tree/carbon-0.6
[15]: https://github.com/carbon-io-examples/contacts-service-advanced/tree/carbon-0.6#generating-api-documentation-aglio-flavor

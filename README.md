# Gluecon, May 24th 2017, Boulder CO

## (0) Hello (D)

* I'm Will Shulman, CEO and Co-founder of mLab.
* Database-as-a-Service for MongoDB.
* mLab hosts more than half a million MongoDB deployments.
* Our infrastructure is large and complex, and is made up of dozens of types of microservices.
* We built carbon.io to allow us to quickly build high-quality microservices.

## (1) What is carbon.io? (D)

* A Node.js framework for building commandline programs, microservices, and APIs.
* It is a framework, build on a set of core libraries.
* It is opinionated.

## (2) Design goals (D)

* Reduce boilerplate code and make it clear how to do the basics (e.g how to structure your application, define endpoints, validate parameters, etc...).
* Seamless integration with MongoDB (although one can use other dbs as well).
* Make control flow a non-issue (a.k.a. avoid callback hell).
* Easy authentication and access control.
* Easy to communicate with other microservices (and therefore easier to build distributed systems).
* Easy to document.
* Easy to test.

## (3) The Basics

### (3.1) Package structure (D)

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

### (3.2) Services, Endpoints, and Operations (D)

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

* Let's take a look at a real example: [Hello world example](https://github.com/carbon-io/example__hello-world-service)

### (3.3) Parameter parsing and input / output validation, and error handling (D)

Operations can be decorated with structure that allows the system to automatically handle certain aspects of managing
inputs and outputs, and makes the API self-describing. 

* Operations can formally define parameters (*query*, *header*, *path*, and *body* parameters).
* Operations can formally define responses by HTTP status code.
* Parameter and response definitions can define JSON Schemas that are automatically enforced.
* This is useful for:
  * Automatic validation of input parameters and response output
  * Generating API documentation
* Let's take a look: [Hello world (advanced) example](https://github.com/carbon-io/example__hello-world-service-advanced).

### (3.4) Working with MongoDB (D)

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

* Part of carbon.io
* Wrapper around standard Node.js MongoDB driver
* Supports synchronous call style XXX

#### (3.4.3) Putting it all together

Let's take a look at a real example that uses MongoDB and that illustrates a few other advanced features:

* [Hello world (mongodb)](https://github.com/carbon-io/example__hello-world-service-advanced-mongodb)


### (3.5) Working with other services (D)

Carbon.io makes it very easy to write services that talk to other services.

* Let's  take a look: [Hello world (chaining)](https://github.com/carbon-io/example__hello-world-service-advanced-chaining)

## (4) A step back: understanding Carbon's core operators (```o```, ```_o```, and ```__```)

The carbon.io framework is built on top of a set of core libraries that, together, provide carbon.io with 
much of its power. 

You will often see carbon.io modules follow this general pattern:

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

### (4.1) Atom (the ```o``` operator) (D)

Atom is the universal object factory, and used to instantiate objects and to create *components*. Components are simply objects bound 
in the Node.js module namespace via ```module.exports```.

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

where ```Rectangle``` and ```Point``` are JavaScript classes with methods you can then invoke on these objects. 

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

JavaScript allows for dynamics properties (via getters and setters) to be defined on objects via ```Object.defineProperty```. Atom 
has a shorthand for this via the ```$property``` keyword:

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

#### (4.1.4) ```o.main```

* Special variant of the ```o``` operator.
* Calls ```_main``` iff ```require.main === module``` (i.e. the module is called as the main module). 
* This allows us to create modules that can act both as applications *and* libraries (subtle point -- pretty cool).
   * We will see example of this with the test framework

#### (4.1.5) Advantages of using Atom:

* Provide for a form of light weight depedency injection (a.k.a. DI).
* Provides for a generic form of application configuration.
* Declarative style ("what not how") via  "datastructure DSLs".

### (4.2) Bond (the ```_o``` operator) (D)

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

#### (4.2.3) Connecting to mongodb via a MongoDB URI
```node
var mydb = _o('mongodb://localhost:27017/mydb')
mydb.getCollection("users").insert({email: "joe@acme.com"))
```

#### (4.2.4) Resolving other Node.js modules (like ```require```)
```node
_o('./HelloEndpoint')
```

#### In previous examples:
* [For environment variables](https://github.com/carbon-io/example__hello-world-service-advanced-mongodb/blob/master/lib/HelloService.js#L51)
* [For organizating modules](https://github.com/carbon-io/example__hello-world-service-advanced-mongodb/blob/master/lib/HelloService.js#L57-L58)
* [For connecting to other services](https://github.com/carbon-io/example__hello-world-service-advanced-chaining/blob/master/lib/PublicHelloService.js#L32) 

### (4.3) Fibers (the ```__``` operator)

Carbon.io uses [Node Fibers](https://github.com/laverdet/node-fibers) under the hood to manage the complexity 
of Node.js concurrency. *Have you noticed any callbacks in the example code so far?*

Consider the following:

```node
fs.readFile(path, function(err, data) {
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
    var data = fs.readFile.sync(path) // control flow is yielded
    console.log(data)
  } catch (err) {
    console.log(err)
  } 
})
```

#### (4.3.1) ```__```

The ```___``` operator is what spawns new fibers.

```node
__(function() {
  // Code that runs in fiber
})
```

you can also optionaly supply a callback:

```node
__(function() {
  // Code that runs in fiber
}, function(err, result) {
  // Code called when fiber returns
})
```

**Behavior**
* Code inside of a spwaned fiber runs asynchronous to the code that spawned the fiber
* If callback is supplied return value from function (or exception if thrown) is passed to callback. 
* From within the fiber ```.sync``` can be called to synchronously call functions (without *actually* blocking). 

#### (4.3.2) ```.sync```

There are two forms of ```.sync```:

* *OBJ*.sync.*METHOD*(ARG_0, ...)
* *OBJ*.*METHOD*.sync(ARG_0, ...)

#### (4.3.3) ```__.ensure``` vs ```__.spawn```

There are two variants of the ```___``` operator, ```ensure``` and ```spawn```.

* ```__.ensure```: Only spawns a new fiber if not already executing within a fiber (**default**)
* ```__.spawn```: Always spawns a new fiber

Using ```__.ensure``` is particularly useful as top-level wrappers for applications that you also want to be able 
to use as components / libraries.

A great example of this are unit tests that you might want to both be part of a larger test suite as well as runnable
standalone. 

```node
var carbon = require('carbon-io')
var __     = carbon.fibers.__(module)
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

### (4.4) Revisiting our examples

* Revisit advanced mongodb
* Revisit chainging

## (5) Authentication and access control

* Authentication is about determining who the user is.
* Access control is about determining what the user has permission to do.

### (5.1) Authentication

#### (5.1.1) The Authenticator class

To implement an Authenticator you create an instance of the ```Authenticator``` class:

```node
o({
  _type: carbon.carbond.security.Authenticator,
  authenticate: function(req) {
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
* [Hello world AAC example](https://github.com/carbon-io/example__hello-world-service-advanced-aac)


## (6) Collections

*is this where we show Contacts API?*

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
    doTest: function(context, done) {
       // test implementation
    }
  })
})
```

Tests can be synchronous or asynchronous. 

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
       try {
         assert(1 == 1)
       } catch (e) {
         done(e)
       }
       done()
    }
  })
})
```

## (8) Generating API documentation for your Services
(do we show docgen throughout?)

## (9) Questions?

## (10) Additional Resources

* http://docs.carbon.io
* http://github.com/carbon-io/carbon-io
* https://www.npmjs.com/package/carbon-io
* will@mlab.com






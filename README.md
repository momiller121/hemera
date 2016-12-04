# Hemera
![Hemera](https://github.com/StarpTech/hemera/raw/master/hemera.png)

[![License MIT](https://img.shields.io/npm/l/express.svg)](http://opensource.org/licenses/MIT)
[![Build Status](https://travis-ci.org/hemerajs/hemera.svg?branch=master)](https://travis-ci.org/hemerajs/hemera)
[![NPM Downloads](https://img.shields.io/npm/dt/nats-hemera.svg?style=flat)](https://www.npmjs.com/package/nats-hemera)
[![Coverage Status](https://coveralls.io/repos/github/hemerajs/hemera/badge.svg?branch=master)](https://coveralls.io/github/hemerajs/hemera?branch=master)
[![Gitter](https://img.shields.io/gitter/room/nwjs/nw.js.svg)](https://gitter.im/hemerajs/hemera)

A [Node.js](http://nodejs.org/) microservices toolkit for the [NATS messaging system](https://nats.io)

- __Status:__ Release Candidate

## 📓 Getting Started

Hemera is a small wrapper around the NATS driver. We want to provide a toolkit to develop micro services in an easy and powerful way. We use bloom filters to provide a pattern matching RPC style. You don't have to worry about the transport. NATS is powerful.

With Hemera you have the best of both worlds. Efficient pattern matching to have the most flexibility in defining your RPC's. It doesn't matter where your server or client lives. You can add the same add as many as you want on different hosts to ensure maximal availability. Thanks to the Request Reply pattern you can work with that as if you do a normal http request. The only dependency you have is a single binary of 7MB. Mind your own business NATS do the rest for you:


### Service Discovery
- Any subscription is managed by NATS. You don't need any service discovery. Total location transparency.

### Scalability
- Filtering on the subject name enables services to divide work (perhaps with locality) e.g. `topic:auth:germany`
- Queue group name allow load balancing of services.

### Fault tolerance
- Auto-heals when new services are added.
- Configure cluster mode to be more reliable.

### Load Balancing
- Queue groups are used by default in Hemera.

\+ **Hemera** = _Pattern-driven micro services._

Table of contents
=================

  * [Prerequisites](#prerequisites)
  * [Installing](#installing)
  * [Example](#example)
  * [Writing an application](#writing-an-application)
    * [Define your service](#define-your-service)
    * [Call your service](#call-your-service)
  * [Pattern matching rules](#pattern-matching-rules)
      * [Matched!](#Matched!)
      * [Not matched!](#not-matched)
  * [Error handling](#error-handling)
      * [Reply an error](#reply-an-error)
      * [Error-first-callbacks](#error-first-callbacks)
      * [Handle timeout errors](#handle-timeout-errors)
      * [Fatal errors](#fatal-errors)
      * [Listen on transport errors](#listen-on-transport-errors)
      * [Specify custom timeout per act](#specify-custom-timeout-per-act)
  * [Delegation](#delegation)
      * [Metadata](#metadata)
      * [Context](#context)
  * [Tracing capabilities](#tracing-capabilities)
      * [Get events](#get-events)
  * [Payload validation](#payload-validation)
  * [Plugins](#plugins)
  * [Logging](#logging)
  * [Protocol](#protocol)
  * [Best practice](#best-practice)
      * [Multiple instances of your service](#create-multiple-instances-of-your-service)
      * [Clustering](#create-another-nats-server-and-create-a-cluster)
  * [Introduction to NATS](#introduction-to-nats)
  * [NATS Limits & features](#nats-limits--features)
  * [Monitoring](#monitoring)
  * [Nginx integration for NATS](#nginx-integration-for-nats)
  * [Contributing](#contributing)
  * [Inspiration](#inspiration)

### Prerequisites

[NATS messaging system](https://nats.io)

We use the Request Reply concept to realize this toolkit. [Request Reply](http://nats.io/documentation/concepts/nats-req-rep/)

### Installing

```
npm i nats-hemera
```

### Example

```js
'use strict';

const Hemera = require('hemera');
const nats = require('nats').connect(authUrl);
    
const hemera = new Hemera(nats, { logLevel: 'info' });

hemera.ready(() => {


  hemera.add({ topic: 'math', cmd: 'add' }, (resp, cb) => {

    cb(null, resp.a + resp.b);
  });

  hemera.add({ topic: 'email', cmd: 'send' }, (resp, cb) => {

    cb();
  })


  hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 2, timeout$: 5000 }, (err, resp) => {
    
    console.log('Result', resp);
  });

  //Without callback
  hemera.act({ topic: 'email', cmd: 'send', email: 'foobar@mail.com', msg: 'Hi' });

});
```

### Writing an application

_Add_: Define you implementation.

_Act_: Start a request

_Topic_: The subject to subscribe. **The smallest unit of Hemera**. It's kind of namespace for your service. If you want to scale your service you have to create a second instance of your service. If you just want to scale a method you have to subscribe to a different subject like `math:additions` because any subscriber have to contain the full implementation of the service otherwise you can run into a `PatternNotFound` exception.

#### Define your service
```js
hemera.add({ topic: 'math', cmd: 'add' }, (resp, cb) => {
  cb(null, resp.a + resp.b);
});
```

#### Call your service
```js
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1 }, (err, resp) => {
console.log(resp); //2
});
```

### Pattern matching rules

A match happens when all properties of added pattern matches with the one in the passed object.

#### Matched!
```js
hemera.add({ topic: 'math', cmd: 'add' }, (resp, cb) => {
  cb(resp.a + resp.b)
});
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1 });
```
#### Not matched!
```js
hemera.add({ topic: 'math', cmd: 'add', foo: 'bar' }, (resp, cb) => {
  cb(resp.a + resp.b)
});
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1 });
```

### Error handling

#### Reply an error
```js
hemera.add({ topic: 'math', cmd: 'add' }, (resp, cb) => {
  cb(new CustomError('Invalid operation'));
});
```
#### Error-first-callbacks
```js
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1 }, (err, resp) => {
 err instanceOf CustomError // true
});
```
#### Handle timeout errors
```js
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1 }, (err, resp) => {
 err instanceOf TimeoutError // true
});
```
#### Fatal errors
Fatal errors will crash your server. You should implement a gracefully shutdown and use a process watcher like PM2 to come back in a clear state. Optional you can disable this behavior by `crashOnFatal: false`
```js
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1 }, (err, resp) => {
 throw new Error('Upps');
});
hemera.add({ topic: 'math', cmd: 'add' }, (resp, cb) => {
   err instanceOf FatalError // true
});
```

#### Listen on transport errors
```js
const hemera = new Hemera(nats, { logLevel: 'info' });
hemera.events.on('error', ...)
hemera.events.on('disconnect', ...)
hemera.events.on('connect', ...)
//see NATS driver for more events
```

#### Specify custom timeout per act
```js
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1, timeout$: 5000 }, (err, resp) => {
});
```

### Delegation
#### _* Notice the use of ```this```_

#### Metadata
If you want to transfer metadata to a service you can use the `meta$` property before sending. It will be passed in all nested `act`.
E.g. you can add a JWT token as metadata to express if your action is legitimate.
```js
hemera.add({ topic: 'math', cmd: 'add' }, function (resp, cb) {
    
    //Access to metadata
    let meta = resp.meta$
    
    cb(null, resp.a + resp.b);
});
```
Will set the metadata only for this `act` and all nested `act`
```js
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1, meta$: { a: 'test' } }, function (err, resp) {

   this.act({ topic: 'math', cmd: 'add', a: 1, b: 5 });
});
```
Will set the metadata on all `act`
```js
hemera.meta$.token = 'ABC1234'
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1}, function (err, resp) {
    //or
   this.meta$.token = 'ABC1234';

   this.act({ topic: 'math', cmd: 'add', a: 1, b: 5 });
});
```
#### Context
If you want to set a context across all `act` you can use the `context$` property.
```js
hemera.context$.a = 'foobar';
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1 }, function (err, resp) {

   this.act({ topic: 'math', cmd: 'add', a: 1, b: 5 }, function (err, resp) {
        
       this.context$.a // 'foobar'
   });
});
```
If you want to set a context only for this `act` and all nested `act`
```js
hemera.act({ topic: 'math', cmd: 'add', a: 1, b: 1, context$: 1 }, function (err, resp) {

   this.act({ topic: 'math', cmd: 'add', a: 1, b: 5 }, function (err, resp) {
        
      this.context$ // 1
   });
});
```

### Tracing capabilities

In any act or add you can access the property `this.parentId$` or `this.requestId$` to get information about your current or parent call. You can listen on the `inbound` event to get detail information about you calls this includes response/request timing and durations.

```js
    result: 10
    error: null
    response$: {
      "startTime": 456447212806, //You can check how long did it take to execute your server-side implementation
      "endTime": 456447216611,
      "duration": 3805 //Total time in nanoseconds
    }
    meta$: {
      "traceId": "XBZPK4DHF2HI3FBODLGB1P" //All requests associated with a specific trace id
    },
    request$: {
      "id": "32HDHW7G1WRMNF2B6Y3KOD", //Current request id
      "parentId": "32HDHW7G1WRMU0ZHX9J864", //Previous request id
      "startTime": 456447211853,
      "endTime": 456447216759,
      "duration": 4906, //Total time in nanoseconds
      "pattern": "a:1,b:3,cmd:sub,topic:math" //Current matched pattern
    }
```

#### Get-events

```js
hemera.on('outbound', (msg) => {
  console.log('Outbound', msg)
})

hemera.on('inbound', (msg) => {
  console.log('Inbound', msg)
})
```

Times are represented in nanoseconds.

### Payload validation
Hemera includes a payload validator called [parambulator](https://github.com/rjrodger/parambulator)
But you can also use different validators e.g [Joi example](https://github.com/hemerajs/hemera/blob/master/examples/custom-payload-validator.js)

```js
hemera.add({
    topic: 'math',
    cmd: 'add',
    a: {
      type$: 'number'
    }
  }, (resp, cb) => {

    cb(null, {
      result: resp.a + resp.b
    });
  });
```
Handling
```js
hemera.act({ topic: 'math', cmd: 'add', a: '1' }, function (err, resp) {
        
   err instanceOf PayloadValidationError //true
});
```
### Plugins

```js
let myPlugin = function (options) {

  let hemera = this;

  hemera.add({
    topic: 'math',
    cmd: 'add'
  }, (resp, cb) => {

    cb(null, {
      result: resp.a + resp.b
    });
  });

};

hemera.use({ plugin: myPlugin, attributes: { name: 'myPlugin' }, options: { } })
```

### Logging

Hemera used Pino logger by default but you can also use your own [example](https://github.com/hemerajs/hemera/blob/master/examples/custom-logger.js)

Your custom logger have to support following log levels.
```js
['info', 'warn', 'debug', 'trace', 'error', 'fatal']
```

```js
const hemera = new Hemera(nats, { logLevel: 'info' });

[2016-11-17T21:04:47.608Z] INFO (app/18196 on starptech): ACT
    topic: "math"
    cmd: "add"
    a: 1
    b: 2
[2016-11-17T21:04:47.613Z] INFO (app/18196 on starptech): ACT_RESP
    topic: "math"
    cmd: "add"
    a: 1
    b: 2
    time$: 2
```

### Protocol

Format: JSON

#### Request
```JSON
{
  "pattern": "<object>",
  "meta$": "<object>",
  "request$": "<object>"
}
```
#### Response
```JSON
{
  "result": "<any>",
  "error": "<serialized_error>",
  "meta$": "<object>",
  "response$": "<object>"
}
```
## Best practice

Think in small parts. A topic is like a service. You can define a service like `auth` which is responsible for authenticate users.

This service has actions like:

```js
hemera.add({ topic: 'auth', cmd: 'authenticate' })
hemera.add({ topic: 'auth', cmd: 'passwordReset' })
...
```

#### Create multiple instances of your service.

Now your service is scaled.

```js
node service.js
node service.js
```

#### Create another NATS Server and create a cluster.

Now your service is fault-tolerant.
```js
var servers = ['nats://nats.io:4222', 'nats://nats.io:5222', 'nats://nats.io:6222'];
var nc = nats.connect({'servers': servers});
new Hemera(nc);
```

### Introduction to NATS

https://www.youtube.com/watch?v=NfL0WO44pqc

### NATS Limits & features
[http://nats.io/documentation/faq/](http://nats.io/documentation/faq/)

### Are you the only one who use NATS for microservice architectures?

> The simplicity and focus of NATS enables it to deliver superior performance and stability with a lightweight footprint. It has the potential of becoming the de-facto transport for microservice architectures and event driven systems in this new era.

Asim Aslam, Creator of Micro

> "I discovered NATS for its performance, and stayed for its simplicity. It’s been a wonderfully reliable layer that connects our microservice architecture at Pressly. The source is easily readable and approachable, and the future developments keep me excited!

Peter Kieltyka - CTO, Pressly

## Running the tests

Set the path to the `gnatsd` before start testing.
```
npm run test
```

## Monitoring

Easy and beautiful tool to monitor you app. [hemera-board](https://github.com/hemerajs/hemera-board)

## Nginx integration for NATS

[nginx-nats](https://github.com/nats-io/nginx-nats)

## Built With

* [Bloomrun](https://github.com/mcollina/bloomrun) - A js pattern matcher based on bloom filters
* [Node Nats Driver](https://github.com/nats-io/node-nats) - Node.js client for NATS, the cloud native messaging system.

## Contributing

Please read [CONTRIBUTING.md](https://gist.github.com/PurpleBooth/b24679402957c63ec426) for details on our code of conduct, and the process for submitting pull requests to us.

## Versioning

We use [SemVer](http://semver.org/) for versioning. For the versions available, see the [tags on this repository](https://github.com/your/project/tags). 

## Authors

* **Dustin Deus** - *Initial work* - [StarpTech](https://github.com/StarpTech)

See also the list of [contributors](https://github.com/StarpTech/hemera/contributors) who participated in this project.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details

## Inspiration

[Seneca](https://github.com/senecajs/seneca) - A microservices toolkit for Node.js.

---
published: False
title: "Part 1: Introduction and Setup"
description: "tags: nestjs, nest, faye, microservices, node.js"
series: "Advanced NestJS Microservices"
cover_image:
canonical_url:
---

*John is a member of the NestJS core team*

xxx longdash:
&#8212;

### Introduction

This article series covers the topic of building a custom **transporter** for the [NestJS microservices subsystem]([xxx](https://docs.nestjs.com/microservices/basics)). If you haven't already read it, please check out my [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series, where I cover many of the basics of the NestJS microservices subsystem architecture, and establish the terminology used in all my Nest Microservices articles.  I'll assume a good understanding of those basics in this article series.

Let's start with the *Why?* question.  The Nest microservices package provides a communications layer abstraction that makes it easy for applications (both Nest and non-Nest) to communicate over a wide variety of what are called **transporters**.  This provides several key benefits, covered fully in the previous article series. Nest comes with a variety of built-in transporters, including NATS, RabbitMQ, Kafka, and others. But what if you want to build your own transporter &#8212; say for ZeroMQ, Google Cloud Pub/Sub, Amazon Kinesis or Apache ActiveMQ? If that's your desire, you've landed on the right article!  Or, if you just want to understand more about how the "magic" works, read on!

A related question is *How can I extend the capabilities of existing Nest transporters?*  For example, maybe you'd like to use the [QoS feature](https://github.com/mqttjs/MQTT.js/blob/master/README.md#qos) with the [MQTT](https://docs.nestjs.com/microservices/mqtt) transporter.  That's a topic I'll cover in another upcoming series (already under development!), but this series provides a baseline for addressing that requirement.

### Article Series Overview

Part 1
Part 2
...

Let's get started!

### Source Code

Working examples and all source code is available [here](https://github.com/johnbiundo/xxx). The articles work best if you follow along with the repository.  Full installation instructions, along with many other details, are available [here](https://github.com/johnbiundo/#xxx).

### What We're Building

We're going to build a custom transporter for the [Faye](https://faye.jcoglan.com/) message broker.  Here's what the final result looks like.  To run this yourself, clone [the repository](xxx) and follow [these instructions]().


![Faye Transporter Demo](./assets/faye-demo.gif 'Faye Transporter Demo')
<figcaption><a name="screen-capture-1"></a>Screen Capture 1: Faye Transporter Demo</figcaption>

xxx
*Description of tmux panes*

### Introduction to Faye

For this case study, we're building our custom transporter to work with the [Faye](https://faye.jcoglan.com/) message broker.  This is a simple OSS JavaScript publish/subscribe message broker that runs nicely on Node.js.  Getting it up and running, and understanding its API, are simple, yet it provides [all the features Nest needs](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3#broker-message-protocol) to build a transporter.

Faye has a very simple &#8212; virtually canonical &#8212; publish/subscribe protocol, as depicted in Figure 1 below.


![Faye Message Protocol](./assets/faye-protocol.png 'Faye Message Protocol')
<figcaption><a name="figure1"></a>Figure 1: Faye Message Protocol</figcaption>

As we've done in the [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series, we'll start by building simple native requestor and responder apps that exercise the Faye API directly.  This will help us make sure we understand the Faye API, and give us a convenient testbed.  To run the code in this section, [read these instructions]() for the github repository.

As covered in the previous article, one of the challenges the Nest microservices package deals with is to layer a request/response message style **on top of** publish/subscribe semantics.  In other words, Nest requestors need to be able to run code like:

```ts
  @Get('customers')
  getCustomers(): Observable<any> {
    this.logger.log('client#send -> topic: "get-customers"');
    // the following request returns a response as an observable
    return this.client.send('get-customers', {});
  }
```

Since Faye (as well as most brokers) only supports publish/subscribe, we need to follow a simple recipe to provide request/response semantics.  Here's a recap of the *generic pattern* for how this is often done.

Let's say component A wishes to "get customers" from component B, which has access to a customer DB. Component A can publish a "get customers" message, and (assuming it has subscribed to that topic) component B receives it, queries the customer DB for a list of customers, and sends a response message. The response message is where the magic happens. In order for B to respond to A, they both must do a few things, agreed upon by convention:

- A chooses a **response topic** (sometimes called a **reply subject**)
- A subscribes to the response topic
- A passes the response topic as part of the initial message
- B uses the response topic as the topic of its own subsequent response message

As we'll see in developing our Faye transporter, Nest makes this a bit easier by automatically choosing the response topic name for you. In fact, Nest builds **two** topics from each **pattern** you declare.  For example, assume you define a Nest microservice responder message pattern like this:

```ts
@MessagePattern('get-customers')
```

(**Note**: Nest uses the term **channel** as a generic internal name for topics (also called subjects and channels by some brokers), and to help disambiguate them from the "user land" concept of a **pattern**, from which they are derived.)

Internally, Nest builds two channels from the above pattern:

* `'get-customers_ack'` - this is the physical channel name we'll use in the Faye transporter to publish/subscribe to **request** messages on the topic `'get-customers'`
* `'get-customers_res'` - this is the physical channel name we'll use in the Faye transporter to publish/subscribe to **response** messages resulting from `'get-customers'` requests

Let's build those native apps we mentioned.  First the responder.

#### Native Responder App (customerService)

Assuming you're [following along](xxx) on the `main` branch, open the file `customerService/src/service.ts`.

Here's the implementation of the `getCustomers` callback handler.  This is the code that is registered to run when our app receives an inbound message on the `'/get-customers_ack'` channel:

```ts
// src/service.ts
function getCustomers(packet): void {
  const message = parsePacket(packet);
  console.log(
    `\n========== <<< 'get-customers' message >>> ==========\n${JSON.stringify(
      message,
    )}\n=============================================\n`,
  );

  // filter customers list if there's a `customerId` param
  const customers =
    message.data && message.data.customerId
      ? customerList.filter(
          cust => cust.id === parseInt(message.data.customerId, 10),
        )
      : customerList;

  client.publish('/get-customers_res', getPayload({ customers }, message.id));
}
```

The logic should be easy to understand. Refer to the full listing and make sure you see how we are using the `'get-customers_res'` and `'get-customers_req'` channels here.  We'll be seeing a lot of that pattern, so make sure it makes sense to you.

One detail to point out is the call to `getPayload()`.  Let's discuss that.  We're going to build these native apps so that they use the internal Nest message format (this topic is covered extensively in the [NestJS Microservices in Action]([xxx](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3)) series).  In this way, they provide a test bench for the Nest transporter and `ClientProxy` we're about to build.  This means we are going to wrap responses in an object we can depict like this:

   ```typescript
   {
     /**
      *  Request error message, if any
      */
     err: any,
     /**
      *  The response message payload (return value of a
      *  message pattern handler)
      */
     response: any
     /**
      *  Status of an observable response. False once the final
      *  stream value has been retrieved.
      */
     isDisposed: boolean
     /**
      *  Unique identifier, corresponding to the id field received
      *  from the initial request
      */
     id: string
   }
   ```

   So `getPayload()` is a helper that wraps our payloads in this structure. If you're paying close attention, you'll notice that we copy the `message.id` field from the inbound request (which also has the canonical Nest message structure) to the outbound field.  We haven't really discussed these `id` fields yet, but they'll become very important soon.

#### Native Requestor App (customerApp)

The requestor is in `customerApp/src/customer-app.ts`.  Below is the implementation of the `getCustomers()` function.

There's a little bit of boilerplate, including wrapping the body in a Promise, which will come in handy later when we want to orchestrate a test with multiple outstanding requests.  But the main logic should be easy to see.  We first subscribe to `'/get-customers_res'` (our **response channel**), simply printing out the result when a message on this channel comes back.  Then we publish the request on `'/get-customers_ack'`.

Because we're running this as a standalone node client program and want to exit after sending a request, we wait 500ms after publishing the request, then cancel the subscription and exit.  Cancelling the subscription is important housekeeping &#8212; if we don't, subscriptions hang around (though Faye will ultimately clean them up).

Finally, you'll notice that much like the responder, we have a helper utility `getPayload()` to wrap our requests in the nest message protocol. You'll also notice that we are generating our `id` field &#8212; the same `id` field we are handling on our responder app above.  For now, just think of the `id` field as something we need to have in order to speak native Nest message protocol.  Later, we'll dive into this in more detail.

```ts
// src/customer-app.ts
async function getCustomers(customerId, requestId = 0, requestDelay = 0) {
  // build Nest-shaped message
  const payload = getPayload(
    '/get-customers',
    customerId
      ? { customerId, requestId, requestDelay }
      : { requestId, requestDelay },
    uuid(),
  );

  return new Promise((resolve, reject) => {
    // subscribe to the response message
    const subscription = client.subscribe('/get-customers_res', result => {
      // handle either objects or stringified results since Nest stringfies,
      // but Faye client lib automatically serializes/deserializes objects
      const parsedResult = parseResult(result);

      console.log(
        `==> Receiving 'get-customers' reply (request: ${requestId}): \n${JSON.stringify(
          parsedResult.response,
          null,
          2,
        )}\n`,
      );
    });

    // once response is subscribed, publish the request
    subscription.then(() => {
      console.log(
        `<== Sending 'get-customers' request with payload:\n${JSON.stringify(
          payload,
        )}\n`,
      );
      const pub = client.publish('/get-customers_ack', payload);
      pub.then(() => {
        // wait .5 second to ensure subscription handler executes
        // then unsubscribe and resolve
        setTimeout(() => {
          subscription.cancel();
          resolve();
        }, 500);
      });
    });
  });
}
```

For completeness, let's take a very quick look at how we run the Faye server. Open up `faye-server/server.js`.  The code is shown below.  You can read more about running the Faye server as a node server [here]([xxx](https://faye.jcoglan.com/node.html)), but we're basically just starting it up, and listening for a series of events that we can log to the console to make it easy to trace the handshaking and message exchange. You shouldn't have to mess with this at all, but if you do, it's pretty basic stuff.

```ts
const http = require('http');
const faye = require('faye');

const mountPath = '/faye';
const port = 8000;

const server = http.createServer();
const bayeux = new faye.NodeAdapter({ mount: mountPath, timeout: 45 });

bayeux.attach(server);
server.listen(port, () => {
  console.log(
    `listening on http://localhost:${port}${mountPath}\n========================================`,
  );
});

bayeux.on('handshake', clientId =>
  console.log('^^ client connect (#', clientId.substring(0, 4), ')'),
);

bayeux.on('disconnect', clientId =>
  console.log('vv client disconnect (#', clientId.substring(0, 4), ')'),
);

bayeux.on('publish', (clientId, channel, data) => {
  console.log(
    `<== New message from ${clientId.substring(0, 4)} on channel ${channel}`,
    `\n    ** Payload: ${JSON.stringify(data)}`,
  );
});

bayeux.on('subscribe', (clientId, channel) => {
  console.log(
    `++ New subscription from ${clientId.substring(0, 4)} on ${JSON.stringify(
      channel,
    )}`,
  );
});

bayeux.on('unsubscribe', (clientId, channel) => {
  console.log(
    `-- Unsubscribe by ${clientId.substring(0, 4)} on ${JSON.stringify(
      channel,
    )}`,
  );
});
```

We've now completed our requestor and responder, and can test them out.  While I only reviewed the `'get-customers'` message above, the code also implements the `'add-customer'` message.  Run the code now by following [these instructions]().  Here's what it looks like:

![Native App Demo](./assets/native-app-demo.gif 'Native App Demo')
<figcaption><a name="screen-capture-2"></a>Screen Capture 2: Native App Demo</figcaption>

### What's Next

In [Part 2](xxx), we'll write a basic version of the Nest transporter's **server** component.  This is the component that runs in the context of a **microservice listener** to enable apps to function as Nest responders (these terms and concepts are discussed extensively in the [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series).

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.

=====================
Probably Part 2, but not clear yet
=====================

### Organizing the Code

Even though the scope of this tutorial is relatively small &#8212; at the end of the day we'll just be creating a few classes &#8212; we're going to be accumulating a number of assets along the way to test and exercise our code. It's time to think a little bit about organization.  First, let's take a look at what we'll end up with:

* Our Faye transporter is going to consist of some interfaces and constants, some serializers (discussed extensively in the [previous article series]()), and the main components: a **client** component (class) providing the `ClientProxy` subclass we'll use in our Nest requestor apps and a **server** component we'll use in our Nest responder apps.
* The native apps we've already written that let us explore the Faye API and give us some test drivers.
* A pair of regular Nest apps: a Nest requestor that is a simple Nest HTTP app called `nestHttpApp` and a Nest responder that is a **microservice**, called `nestMicroservice`.  These are the apps that will **use** our new Faye transporter.

Where to keep all of these assets?

The main thing to think about now is where to keep the Faye transporter components.  Since we'll want to easily re-use them across different Nest apps, the logical place is an NPM package.  To that end, from this point forward when we work on any of those components, they'll go into that package.

#### The Custom Transporter Package

At this point, you should check out the `part2` branch of the repo that you cloned.  [Read more here](xxx) for details.  Once you've checked out that branch, you'll notice a new folder called `nestjs-faye-transporter`.  This is organized as an NPM package (read more about creating NestJS-friendly NPM packages in my article []()).

Here's a run-down on the contents of that package, and how we'll use it.

1. The scripts (`package.json`) let you build (`npm run build`), run (`npm run build`) and, if you want, publish (`npm publish`) the package.
2. Our workflow, in dev, will be to build the package and then use `npm link` to install it in our Nest apps.  More on this soon.
3. The code is organized in a directory structure as follows:
     * Code common to our **client** and **server** is in top level files or folders, like `src/external` (holds things like interfaces to the Faye client library which are *external* to the Nest environment), `src/interfaces` (holds interfaces used within the Nest constructs), and `src/constants`.
     * All code defining the **client** is in the `src/requestor` folder. Ultimately, this will consist of a sub-class of `ClientProxy` from the `@nestjs/microservices` package &#8212; the class which provides things like `client.send(...)` and `client.emit()`) &#8212; and a variety of related classes.
     * All code defining the **server** is in the `src/responder` folder.  Ultimately, this will consist of a sub-class of `Server` from the `@nestjs/microservices` package &#8212; the class which implements a transporter strategy &#8212; and a variety of related classes.

### First Iteration of the Server Component

Let's get started building the "server" side of the equation -- the part of the custom transporter that you'll use to build Nest responders (see [xxx]() for more on this terminology, but think of a Nest responder as a Nest application that listens for inbound requests over a specific transport, like Faye).

Again, for this step of the tutorial, we should be on the `part2` branch.

#### Using a Custom Transporter Strategy

Before we start writing transporter code, let's first take a look at how we're going to use it.  We'll plug it into the by-now-familiar `nestMicroservice` app.  The code for this app is in the top-level `nestMicroservice` folder.

Open the `main.ts` file.  Notice the structure of the `createMicroservice` call:

```ts
const app = await NestFactory.createMicroservice(AppModule, {
  strategy: new ServerFaye({
    url: 'http://localhost:8000/faye',
    retry: 5,
    timeout: 120,
    serializer: new OutboundResponseIdentitySerializer(),
    deserializer: new InboundMessageIdentityDeserializer(),
  }),
});
```

This is very similar to the structure used for any built-in Nest transporter.  For example, with MQTT it looks like:


```ts
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    host: 'localhost',
    port: 1883,
    serializer: new OutboundResponseIdentitySerializer(),
    deserializer: new InboundMessageIdentityDeserializer(),
  },
});
```

It should be clear what we're doing: instead of passing the normal [transporter options object](https://docs.nestjs.com/microservices/basics#getting-started) with `transport` and `options` properties, we pass a single property, `strategy`, whose value is an **instance** of our custom transporter class.

#### Take One

Alright, we're finally ready to look at our `ServerFaye` class &#8212; the one we'll instantiate and pass as that `strategy` property above.  We'll take this in a few steps.  Start by opening the file `nestjs-faye-transporter/src/responder/transporters/server-faye.ts`.

This is going to be the absolute bare-bones class needed to implement the Faye transporter.  We're going to make a number of simplifications, so that we can focus on the core flow.  Our first cut will:

- minimize error handling
- not rely on some nice features of the framework that make our code really robust
- not handle events (e.g., inbound messages coming from `client.emit(...)`)
- not really be type safe (we omit a bunch of typing to declutter the code)

To state it in more direct terms, our requirement for this cut is to simply be able to respond to an inbound **request** (in the sense of a request from a request-response style message).  We'll test this requirement by replacing our native `customerApp` from the last chapter with our `nestMicroservices` app running our new custom transporter.


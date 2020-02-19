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

This article series covers the topic of building a custom **transporter** for the [NestJS microservices subsystem](xxx). If you haven't already read it, please check out my [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series, where I cover many of the basics of the NestJS microservices subsystem architecture, and establish the terminology used in all my Nest Microservices articles.  I'll assume a good understanding of those basics in this article series.

Let's start with the *Why?* question.  The Nest microservices package provides a communications layer abstraction that makes it easy for applications (both Nest and non-Nest) to communicate over a wide variety of what are called **transporters**.  This provides several key benefits, covered fully in the previous article series. Nest comes with a variety of built-in transporters, including NATS, RabbitMQ, Kafka, and others. But what if you want to build your own transporter &#8212; say for ZeroMQ, Google Cloud Pub/Sub, Kinesis or ActiveMQ? If that's your desire, you've landed on the right article!  Or, if you just want to understand more about how the "magic" works, read on.

A related question is *How can I extend the capabilities of existing Nest transporters?*  For example, maybe you'd like to use the [QoS feature]() with the [MQTT]() transporter.  That's a topic I'll cover in another upcoming series (already under development!), but this series provides a baseline for addressing that requirement.

### Article Series Overview

Part 1
Part 2
...

Let's get started!

### Source Code

Working examples and all source code is available [here](https://github.com/johnbiundo/xxx). The articles work best if you follow along with the repository.  Full installation instructions, along with many other details, are available [here](https://github.com/johnbiundo/#xxx).

### What We're Building

We're going to build a custom transporter for the[Faye](https://faye.jcoglan.com/) message broker.  Here's what the final result looks like.  To run this yourself, clone [the repository](xxx) and follow [these instructions]().


![Faye Transporter Demo](./assets/faye-demo.gif 'Faye Transporter Demo')
<figcaption><a name="screen-capture-1"></a>Screen Capture 1: Faye Transporter Demo</figcaption>

xxx
*Description of tmux panes*

### Introduction to Faye

For this case study, we'll build our custom transporter to work with the [Faye](https://faye.jcoglan.com/) message broker.  This is a simple OSS JavaScript publish/subscribe message broker that runs nicely on Node.js.  Getting it up and running, and understanding its API, are simple, yet it provides [all the features Nest needs](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3#broker-message-protocol) to build a transporter.

Faye has a very simple &#8212; virtually canonical &#8212; publish/subscribe protocol, as depicted in Figure 1 below.


![Faye Message Protocol](./assets/faye-protocol.png 'Faye Message Protocol')
<figcaption><a name="figure1"></a>Figure 1: Faye Message Protocol</figcaption>

As we've done in the [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series, we'll start by building simple native requestor and responder apps.  To run the code in this section, [read these instructions]() for the github repository.

As covered in the previous article, one of the challenges Nest microservices deals with is to layer a request/response message style **on top of** publish/subscribe semantics.  In other words, Nest requestors need to be able to run code like:

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

Nest makes this a bit easier by automatically choosing the response topic name for you. In fact, Nest builds **two** topics from each **pattern** you declare.  For example, assume you define a Nest microservice responder message pattern like this:

```ts
@MessagePattern('get-customers')
```

Nest uses the term **channel** as a generic internal name for topics (also called subjects and channels by some brokers), and to help disambiguate them from the "user land" concept of a **pattern**.  So, internally, Nest builds two channels from the above pattern:

* `'get-customers_ack'` - this is the physical channel/topic/subject name we'll use in the Faye transporter to publish/subscribe to **request** messages
* `'get-customers_res'` - this is the physical channel/topic/subject name we'll use in the Faye transporter to publish/subscribe to **response** messages

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

The logic should be easy to understand.  One thing to point out is the call to `getPayload()`.  Let's discuss that.  We're going to build these native apps to speak the internal Nest message protocol (this topic is covered extensively in the [NestJS Microservices in Action](xxx) series) so they provide a test bench for the Nest transporter and `ClientProxy` we're about to build.  This means we are going to wrap responses in an object we can depict like this:

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

   So `getPayload()` is a helper that wraps our payloads in this structure. If you're paying close attention, you'll notice that we copy the `message.id` field from the inbound request (which also has the canonical Nest message protocol structure) to the outbound field.  We haven't really discussed these `id` fields yet, but they'll become very important soon.

#### Native Requestor App (customerApp)

The requestor is in `customerApp/src/customer-app.ts`.  Below is the implementation of the `getCustomers()` function.

There's a little bit of boilerplate, including wrapping the body in a Promise, which will come in handy later when we want to orchestrate a test with multiple outstanding requests.  But the main logic should be easy to see.  We first subscribe to `'/get-customers_res'` (our **response channel**), simply printing out the result when a message on this channel comes back.  Then we publish the request on `'/get-customers_ack'`.

Because we're running this as a standalone node client program and want to exit after sending a request, we wait 500ms after publishing the request, then cancel the subscription and exit.  Cancelling the subscription is important housekeeping &#8212; if we don't, subscriptions hang around (though Faye will ultimately clean them up).

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

For completeness, let's take a very quick look at how we run the Faye server. Open up `faye-server/server.js`.  The code is shown below.  You can read more about running the Faye server as a node server [here](xxx), but we're basically just starting it up, and listening for a series of events that we can log to the console to make it easy to trace the handshaking and message exchange.

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

We've now completed our requestor and responder, and can test them out.  Do this by following [these instructions]().  Here's what it looks like:

![Native App Demo](./assets/native-app-demo.gif 'Native App Demo')
<figcaption><a name="screen-capture-2"></a>Screen Capture 2: Native App Demo</figcaption>

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
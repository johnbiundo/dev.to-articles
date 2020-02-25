---
published: False
title: "Part 1: Introduction and Setup"
description: "tags: nestjs, nest, faye, microservices, node.js"
series: "Advanced NestJS Microservices"
cover_image: "https://dev-to-uploads.s3.amazonaws.com/i/lgt7m4a8dton9rvuimje.gif"
canonical_url:
---

*John is a member of the NestJS core team*

### Introduction

This article series covers the topic of building a **custom transporter** for the [NestJS microservices subsystem](https://docs.nestjs.com/microservices/basics). If you haven't already read it, please check out my [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series, where I cover many of the basics of the NestJS microservices subsystem architecture, and establish the terminology used in all my Nest Microservices articles, including answering the question *"what the heck is a transporter?"*.  I'll assume a good understanding of those basics in this article series.

Let's start with the *Why?* question.  The Nest microservices package provides a communications layer abstraction that makes it easy for applications (both Nest and non-Nest) to communicate over a wide variety of what are called **transporters**.  This provides several key benefits, covered fully in the [previous article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3). Nest comes with a variety of built-in transporters, including NATS, RabbitMQ, Kafka, and others. But what if you want to build your own transporter &#8212; say for ZeroMQ, Google Cloud Pub/Sub, Amazon Kinesis or Apache ActiveMQ? If that's your desire, you've landed on the right article!  Or, if you just want to understand more about how the "magic" works, read on!

A related question is *How can I extend the capabilities of existing Nest transporters?*  For example, maybe you'd like to use the [QoS feature](https://github.com/mqttjs/MQTT.js/blob/master/README.md#qos) with the [MQTT](https://docs.nestjs.com/microservices/mqtt) transporter.  That's a topic I'll cover in another upcoming series (already under development!), but this series provides a baseline for addressing that requirement.

### Article Series Overview

* Part 1 (this article): Sets the background for the entire series.  We'll begin by examining the simple [Faye message broker](https://faye.jcoglan.com/) to set the stage for building a custom Nest transporter that sits on top of it.
* [Part 2](https://dev.to/nestjs/part-2-basic-server-component-5313-temp-slug-6221883?preview=2f3ceab6d03c32bc1d00e56a907f4c2e87b388b516d6009c5c72a6f5a31ef8da2a310c035b7b0a84cd9760ab2ac5d241dd2ceaceaf807ba1e745bbb9): Walks you through building an initial version of the **server side** of a transporter. The result is a working version that should familiarize you with the main concepts needed to understand how to build the server component of **any** custom transporter. It defers implementation of a few important details to the next article so as to keep us focused on the main path.
* [Part 3](https://dev.to/nestjs/part-3-completing-the-server-component-2fai-temp-slug-8783531?preview=be5cb28367d68473fba3e9a91c71084b83414317c27529045d1732b885da4cedb2020d8a7a32482e950f79db2908dee597c475f0f0b1a77bb73f0cab): Cleans up some loose ends, and deep dives on a few of the more advanced and impressive features of the framework, and shows you how to enable these features in your custom transporter.
* [Part 4](https://dev.to/nestjs/part-4-basic-client-component-298b-temp-slug-9977921?preview=21ec3d333fc6d9d92c11dcbd8430a5132e93390de84cb4804914aa143492e925e4299ca3eb7f376918c1ed77df56e29db2572e5d6f7ab235b3e5f2b9): Switches to the client side. As with the server side, this article walks through a **complete** implementation, though it defers a few of the more complex features to the next article.
* [Part 5](https://dev.to/nestjs/part-5-completing-the-client-component-hlh-temp-slug-2907984?preview=82c11163db963ca01d8d62d3a7b14843b422a6b28f46762d999bbe4b7035ad634d48bbbdd740e36376121aa673354ff5259f8b3028bceb931e800d9e): Completes the client-side journey, resulting in a completely functional custom transporter for Faye.
* Part 6: (Coming soon!) Surveys a few of the built-in NestJS transporters and compares their implementation to the Faye implementation to shed light on some broker-specific nuanced implementation details.

Let's get started!

#### Get the Code

All of the code in these articles is available [here](https://github.com/johnbiundo/nestjs-faye-transporter-sample).  As always, these tutorials work best if you follow along with the code.  The [README](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md) covers all the steps you need to get the repository, build the apps, and follow along.  It's easy!  I strongly encourage you to do so.  Note that each article has a corresponding branch in the repository.  For example, this article (part 1), has a corresponding branch called `part1`. Read on, or [get more details here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#repository-structure) on using the repository.

#### Git checkout the current version

For this article, you should `git checkout` the branch `part1`.  You can get more information [about the git repository branches here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#repository-structure).

#### Build the Apps for This Part

For each article in the series, we introduce some new components (and sometimes entirely new **projects**).  For convenience, at the start of each article, you should run the command\*

```bash
$ sh build.sh
```
\**This is a script that runs on Linux-like systems.  You may have to make an adjustment for non-Linux systems.  Note that the script is simply a convenience that runs `npm install` inside each top-level directory, so you can always fall back to that technique if you have trouble with the script.*

### Overview for Part 1

For this case study, we'll build a custom transporter to work with the [Faye](https://faye.jcoglan.com/) message broker.  This is a simple OSS JavaScript publish/subscribe message broker that runs nicely on Node.js.  Getting it up and running, and understanding its API, are simple, yet it provides [all the features Nest needs](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3#broker-message-protocol) to build a transporter.

Faye has a very simple &#8212; virtually canonical &#8212; publish/subscribe protocol, as depicted in Figure 1 below.

![Faye Message Protocol](./assets/faye-protocol2.png 'Faye Message Protocol')
<figcaption><a name="figure1"></a>Figure 1: Faye Message Protocol</figcaption>

### Building Basic Faye Client Apps

As we've done in the [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series, we'll start by building simple native requestor and responder apps that exercise the Faye API directly.  This will help us make sure we understand the Faye API, and give us a convenient testbed.

As covered in the [previous article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3), one of the challenges the Nest microservices package deals with is to layer a request/response message style **on top of** publish/subscribe semantics.  In other words, Nest requestors need to be able to run code like:

```ts
  @Get('customers')
  getCustomers(): Observable<any> {
    this.logger.log('client#send -> topic: "get-customers"');
    // the following request returns a response as an observable
    return this.client.send('get-customers', {});
  }
```

#### Supporting Request/Response

Since Faye (as well as most brokers) only supports publish/subscribe, we need to follow a simple recipe to provide request/response semantics.  Here's a recap of the *generic pattern* for how this is often done.

Let's say component A wishes to "get customers" from component B, which has access to a customer DB. Component A can publish a "get customers" message, and (assuming it has subscribed to that topic) component B receives it, queries the customer DB for a list of customers, and sends a response message. The response message is where the magic happens. In order for B to respond to A, they both must do a few things, agreed upon by convention:

- A chooses a **response topic** (sometimes called a **reply subject**)
- A subscribes to the response topic
- A passes the response topic as part of the initial message
- B uses the response topic as the topic of its own subsequent response message

As we'll see in developing our Faye transporter, Nest makes this a bit easier by automatically choosing the response topic name for you. In fact, Nest builds **two** topics from each **pattern** you declare.  For example, assume you define a Nest microservice responder *message pattern* like this:

```ts
@MessagePattern('get-customers')
```

(**Note**: Nest uses the term **channel** as a generic internal name for topics (also called subjects and channels by some brokers), and to help disambiguate them from the "user-land" concept of a **pattern**, from which they are derived.)

Internally, Nest builds two channels from the above pattern:

* `'get-customers_ack'` - this is the physical channel name we'll use in the Faye transporter to publish/subscribe to **request** messages on the topic `'get-customers'`
* `'get-customers_res'` - this is the physical channel name we'll use in the Faye transporter to publish/subscribe to **response** messages resulting from `'get-customers'` requests

Let's build those native apps we mentioned.  **Note**: later we'll want to use these native apps to interact with our Nest transporter, so we'll utilize the Nest channel names described above.  We'll start with the responder app.

#### Native Responder App (customerService)

Assuming you're [following along](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-1-introduction-and-setup) on the `part1` branch, open the file `customerService/src/service.ts`.

Here's the implementation of the `getCustomers` callback handler.  This is the code we registered to run (see the `subscribe()` call lower in the file) when our app receives an inbound message on the `'/get-customers_ack'` channel:

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

One detail to point out is the call to `getPayload()`.  Let's discuss that.  As mentioned, we're building these native apps to conform to Nest protocols.  This includes using the channel names, as well as matching the internal Nest message format (this topic is covered extensively in the [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3)) series).  Because of this, the apps provide a convenient test bench for the Nest transporter and `ClientProxy` we're building in this article series.  The message format requirement means we need to wrap responses in a standard object we can depict like this:

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

So we can recognize that `getPayload()` is a helper that wraps payloads in this standard Nest message structure. If you're paying close attention, you'll notice that `getPayload()` (not shown here &#8212; look at the `service.ts` source code to see it) copies the `message.id` field from the inbound request (which we'll ensure also has the canonical Nest message structure) to the outbound message.  We haven't really discussed these `id` fields yet, but they'll become very important when we dive into the "client side" of our transporter starting in Part 4.

#### Native Requestor App (customerApp)

The requestor is in `customerApp/src/customer-app.ts`.  Below is the implementation of the `getCustomers()` function.

There's a little bit of boilerplate, including wrapping the body in a Promise (which isn't required, but will come in handy later when we want to orchestrate a test with multiple outstanding requests).  But the main logic should be easy to see.  It implements the **requestor** side of that [request/response protocol](#requestresponse) we walked through a couple of minutes ago.

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

We first subscribe to `'/get-customers_res'` (our **response channel**).  The handler simply prints out the result when a message on this channel comes back. This is the "final response" to an end-user command (request) issued from the command line (see the basic command line processing logic at the bottom of the file).

Then we publish the request on `'/get-customers_ack'`.

Because we're running this as a standalone node command line program, and thus want to exit after sending a request (letting us use it as a command line program that we can repeatedly send requests), we wait 500ms after publishing the request, then cancel the subscription and exit.  Cancelling the subscription is important hygiene &#8212; if we don't, subscriptions hang around (though Faye will eventually detect them and clean them up).

Finally, you'll notice that much like the responder, we have a helper `getPayload()` utility to wrap our requests in the Nest message protocol. You'll also notice that we are generating our `id` field &#8212; the same `id` field we are handling on our responder app above.  For now, just think of the `id` field as something we need to have in order to speak native Nest message protocol.  Later, we'll dive into this in more detail.

For completeness, let's take a very quick look at how we run the Faye server. Though you can treat it as a black box, it's worth a moment to look at how it runs, and appreciate its native node.js-ness and simplicity.  Open up `faye-server/server.js`.  The code is shown below.  You can read more about running the Faye server [here](https://faye.jcoglan.com/node.html), but we're basically just starting it up, and for instrumentation purposes, listening for various events that we can log to the console to make it easy to trace the handshaking and message exchange. You shouldn't have to mess with this at all, but if you do, it's pretty basic stuff.

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

We've now completed our requestor and responder, and can test them out.  While I only reviewed the `'get-customers'` message flow above, the code also implements the `'add-customer'` message.  Run the code now by following [these instructions](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-1-introduction-and-setup).  Here's what it looks like. In this video clip, the Faye broker is running in the top pane, the `customerService` app is running in the middle pane, and the `customerApp` app is running in the lower pane (and if you're curious, this is [tmux](https://github.com/tmux/tmux); you can get my setup [here](https://github.com/johnbiundo/nest-nats-sample#pro-tip-use-tmux-optional)).

![Native App Demo](./assets/faye-basic-demo2.gif 'Native App Demo')
<figcaption><a name="faye-basic-demo"></a>Screen Capture: Native App Demo</figcaption>

### What's Next

In [Part 2](https://dev.to/nestjs/part-2-basic-server-component-5313-temp-slug-6221883?preview=2f3ceab6d03c32bc1d00e56a907f4c2e87b388b516d6009c5c72a6f5a31ef8da2a310c035b7b0a84cd9760ab2ac5d241dd2ceaceaf807ba1e745bbb9), we'll write a basic version of the Nest transporter's **server** component.  This is the component that runs in the context of a **microservice listener** to enable apps to function as Nest responders (these terms and concepts are discussed extensively in the [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series).

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
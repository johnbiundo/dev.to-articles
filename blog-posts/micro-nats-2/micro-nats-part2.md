---
published: false
title: "Integrate NestJS with External Services using Microservice Transporters (Part 2)"
description: "tags: nestjs, nest, NATS, microservices, node.js"
series: "NestJS Microservices in Action"
cover_image: "https://dev-to-uploads.s3.amazonaws.com/i/1khd2pysk1o2go9e0oxs.jpg"
canonical_url:
---

*John is a member of the NestJS core team*

### Part 2: Digging into Transporter Communications

This is Part 2 of a series on using Nest microservices as an integration technology. [Part 1](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) lays the foundation with an introduction to the basic communication concepts used by Nest microservices.

This article lays out the concepts and challenges of integrating Nest apps with non-Nest apps.

### Get the Code

Reminder: all the code for these articles is available [here](https://github.com/johnbiundo/nest-nats-sample), with complete instructions [here](https://github.com/johnbiundo/nest-nats-sample#sample-repository-for-nestnatsmicroservice-article-series).

### Roles in Action

In the prior article, I covered [the roles](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3#a-vocabulary-for-transporter-use-cases) a Nest app plays in various communication scenarios. Let's dig further into those roles and how they matter in each integration use case.

#### Nest as Requestor

Let's start with a quick review of Nest's transporter application-level protocol. This is covered in depth [in the Nest documentation](https://docs.nestjs.com/microservices/basics#client), but we'll summarize briefly here, marrying Nest concepts with our freshly minted terminology covered at the end of [Part 1](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3#a-vocabulary-for-transporter-use-cases).

When a Nest component is in the role of [**Nest as requestor**](#nest-as-requestor), it performs its role through an instance of the `ClientProxy` class which has been configured to work with a specific transporter. Such a requestor may be housed in any sort of Nest app. For example, to make a `'get-customers'` request from a Nest HTTP-based app (e.g., from within a REST route handler), via NATS, we would instantiate a `ClientProxy` with code something like this:\*

```typescript
// app.controller.ts
@Client({
  transport: Transport.NATS,
  options: {
    url: 'nats://localhost:4222',
  }
})
client: ClientProxy;
```

\*Note: we use the `@Client()` decorator as a convenience, but Nest recommends using the dependency injection method for creating a `ClientProxy` instance. Both work for our purposes, but the DI method has some advantages for production development. See the [Nest documentation](https://docs.nestjs.com/microservices/basics#client) for more details.

Using this `ClientProxy` instance, we would then add code to our REST route handler (e.g., one that responds to the REST `GET customers` request) to issue the NATS request with code something like this:

```typescript
// app.controller.ts
@Get('customers')
async getCustomers(): Observable<customers> {
  return this.client.send('get-customers', {});
}
```

Let's update the diagram from [Figure 1, Case C](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3#figure1) to reflect this terminology.

![Case C](./assets/case-c.png 'Case C')
<a name="figure1"></a><figcaption>Figure 1: Nest as Requestor</figcaption>

#### Nest as Responder

When a Nest component is in the role of [**Nest as responder**](#nest-as-responder), things are slightly more complex. We need the component to function **inside the context of a network listener** (in order to receive inbound messages from remote senders). This concept gives us an understanding of what we can now refer to as a Nest **microservice**. A Nest microservice is a component that binds some behavior to incoming network messages bearing specific **topics** (and, optionally, **payloads**). Obviously the microservice listener must connect to the correct broker, and possibly be configured with other parameters. Making this association between a microservice listener and a particular broker, and specifying configuration parameters for that association, is the job of the transporter. For example, a Nest component that can respond to a NATS `'get-customer'` message would need several moving parts.

First, we need to start up a network listener. This is covered in detail [in the Nest documentation here](https://docs.nestjs.com/microservices/basics#getting-started), but the code is straightforward if you're familiar with a typical Nest `main.ts` file:

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.NATS,
    options: {
      url: 'nats://localhost:4222',
    },
  });
  app.listen(() => console.log('Microservice is listening...'));
}
```

All the transporter-specific details &#8212; such as how to connect to the NATS broker &#8212; are passed in as an options object to the `createMicroservice()` call.

Then we need to register the handler for any inbound request we want to handle. This is covered in detail [here](https://docs.nestjs.com/microservices/basics#getting-started), but the handler code, running inside a microservice controller, is straightforward if you're familiar with typical Nest HTTP controllers:

```typescript
// app.controller.ts
@MessagePattern('get-customers')
getCustomers(@Payload() data: any) {
  return customerService.getCustomers();
}
```

We can now update [Figure 1 Case B]() to reflect this understanding.

![Case b](./assets/case-b.png 'Case C')
<a name="figure2"></a><figcaption>Figure 2: Nest as Responder</figcaption>

The other roles discussed earlier &#8212; emitter and subscriber &#8212; are similar (and simpler). For brevity, we'll omit the diagrams and just describe the differences. An emitter is like a requestor except it does not expect a response, so it does not subscribe to a response message. Like a requestor, it's issued via a `ClientProxy` instance. A subscriber is like a responder except it does not issue a response message. Like a responder, it's housed as a handler within a microservice listener.

#### Running an All Nest Stack

Full implementations of these apps are included in the repository available [here](https://github.com/johnbiundo/nest-nats-sample). Running these two Nest apps as shown in <a href="#figure1">Case A of Figure 1</a>, which you can do now by following [these instructions](https://github.com/johnbiundo/nest-nats-sample#running-the-all-nest-configuration), is easy. Following those steps will start up an instance of a NATS broker, in verbose mode, which is very helpful for watching the message flow. Take a few minutes to run through the full installation and to examine the code.

Of course so far, we haven't really had to deal with any external app components. This means there are no messy details, and the Nest components "just work". Let's move on to the topic of external app components now.

### External NATS app

To give us something concrete to look at, let's quickly construct basic implementations of our external apps. We'll build two of them: one as a requestor and one as a responder. These will serve as sandboxes for examining live message flow behavior, and play the role of the "external apps" we see in <a href="#figure1">Figure 1</a>.

As native NATS apps, these can immediately and seamlessly communicate with each other.

![external apps](./assets/external-apps-nats.png 'External Apps')
<a name="figure3"></a><figcaption>Figure 3: External NATS Apps</figcaption>

#### The customerApp application

This app functions as a **requestor**, and is quite simple. When the app is called with the command line argument `"get-customers"`, it first connects to the NATS server, then runs the `getCustomers()` function. Here's the essence of that function (slightly simplified, and minus a little logging and error handling).

```typescript
// customer-app.ts
async function getCustomers() {
  const response = await nats.request('get-customers', 1000, {});
  console.log(
    'getCustomers reply: \n',
    JSON.stringify(JSON.parse(response.data), null, 2)
  );
}
```

#### The customerService application

This app functions as a **responder**. The code below is the essence of the `main()` function, which simply starts up, connects to NATS, and subscribes to a couple of topics. **Note:** to stay focused on the important elements, the code below is slightly simplified from the code in the repository, though the overall structure and intent remains the same.

```typescript
// service.ts
async function main() {
  nats = await connect({
    servers: [NATS_URL],
    timeout: 1000,
  });
  console.log('NATS customer service starts...');

  nats.on('connect', (client, url, serverInfo) => {
    nats.subscribe('get-customers', getCustomers);
    nats.subscribe('add-customer', addCustomer);
  });
}
```

The code below is the essence of the `getCustomers()` callback function (again, simplified slightly from the repo code), which has been registered as the handler (line #9 above) to call when the `'get-customers'` message is received.

```typescript
// service.ts
async function getCustomers(err, message): Promise<void> {
  if (err) {
    return;
  }

  if (message.reply) {
      const customers =
        message.data && JSON.parse(message.data).id
          ? customerList.filter(
              cust => cust.id === parseInt(JSON.parse(message.data).id, 10)
            )
          : customerList;
      await nats.publish(message.reply, JSON.stringify({ customers })
  } else {
    console.error('Malformed request.  No reply field included in request.');
  }
}
```

If you look at the full repository source code, you'll see that our "database" is just a stub with an array of customers stored in a local `customerList` variable.

Most importantly, the `getCustomers()` method shows the **request-response** implementation clearly. The logic is straightforward:

1. check to see if a customer id is supplied. If so retrieve that customer, otherwise retrieve all customers.
2. **publish** a response message using the `reply` topic passed in on the request message.  Include the customer list as the payload in that response.

The full source code for these apps is available [here](https://github.com/johnbiundo/nest-nats-sample). Read more about running them [here](https://github.com/johnbiundo/nest-nats-sample#running-the-all-native-app-configuration). I highly encourage you to do this to get familiar with their behavior, and to observe the NATS message flows first-hand.

#### NATS Client Library Native Support for Requests

You may have noticed we made a call to `request()` in the customerApp above, and wondered why (perhaps you were expecting us to call something like `publish()`). Nice catch! Here's a quick geeky aside on that topic that also provides a little insight into the Nest transporter abstraction layer.

If you dig into the NATS client API libraries (for example, the [TypeScript client](https://github.com/nats-io/nats.ts)), you'll notice that they provide API calls to both the NATS `PUB` verb (usually `publish()`) and `SUB` verb (usually `subscribe()`), but also offer a `request()` method that doesn't have a direct counterpart in the NATS protocol (see an example [here](https://github.com/nats-io/nats.ts#basic-usage)). By now, you should probably be able to guess what `request()` does. It uses essentially the same pattern we [described in the previous article](#broker-message-protocol)  &#8212;  implemented by Nest to provide request/response semantics on top of publish/subscribe messaging  &#8212;  **within the NATS TypeScript client library itself**. To be clear: this is a convenience provided by the client library (not a native feature of NATS).

In the case of the NATS transporter, Nest takes advantage of this client library feature directly, rather than emulating it. For other transporters, where no such "request/response abstraction" exists, Nest emulates this functionality. In the end, both Nest and the client libraries share a similar need to add request/response behavior on top of a publish/subscribe model.

### Clash of the Message Formats

#### Different Message Formats

One thing we can do, now that we have an all-Nest version and a native TypeScript/NATS version of the requestor and responder, is examine the actual NATS messages exchanged. You can and should do this yourself (for now, you can examine the NATS logs produced by following [these instructions](https://github.com/johnbiundo/nest-nats-sample#running-the-all-nest-configuration) and [these instructions](https://github.com/johnbiundo/nest-nats-sample#running-the-all-native-app-configuration)), but let's cut right to the chase. Nest encodes message payloads in a format that is probably **not directly compatible with the format used by your external app**. We'll address why that is, and how to resolve the incompatibility, below. Now we're getting to the meaty part of the article series!

**Note**: the following representations of the actual messages as seen in the NATS server log take a few **small** liberties to simplify and clarify. What you'll actually see in the logs is slightly more verbose. To see the NATS log messages yourself, follow [these instructions](https://github.com/johnbiundo/nest-nats-sample#running-the-all-nest-configuration).

#### Native App Messages

The native *customerApp*, when it issues the `'get-customers'` request, emits a message that looks like this:

```bash
PUB get-customers _INBOX.6EADK
MSG_PAYLOAD: {}
```

Here, you see the message topic (`get-customers`) and the reply topic (`_INBOX.6EADK`) on the first line (they're part of the message "header", not the payload), and an empty message payload on the second line.

The *customerApp* receives a response back from *customerService* (because it included a response topic in the request - `_INBOX.6EADK` - that it subscribed to) that looks like this:

```bash
MSG_PAYLOAD: {"customers": [{ "id": 1, "name": "nestjs.com" }]}
```

Here, payload is just a "stringified" version of our JSON response.

#### Nest App Messages

On the other hand, the Nest components produce slightly different messages. The *nestHttpApp*, when it runs `client.send('get-customers', {})`, emits a message that looks like this:

```bash
PUB get-customers _INBOX.9FEAM
MSG_PAYLOAD: {"pattern": "get-customers","data": {},
"id": "84d9259e-fd00-4456-83b8-408311ca72cc"}
```

It receives a response back from *nestMicroservice* that looks like this:

```bash
MSG_PAYLOAD: {"err": null,"response": [{ "id": 1, "name":"nestjs.com" }],
"isDisposed": true,"id": "84d9259e-fd00-4456-83b8-408311ca72cc"}
```

The differences should be clear. Nest wraps your message payloads inside a JSON object. For requests, your payload is wrapped in a `data` property. For responses, your payload is wrapped in a `response` property.

Why the differences? Consider that Nest must properly route and manage the lifetime of messages **within and between Nest apps** (e.g., our "Pure NestJS" <a href="https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3#figure1">Case A in Figure 1</a>). Nest needs to pass some metadata, along with the actual application-specific message content, with each message. Nest encodes this metadata in the **payload** itself (because NATS doesn't allow you to add fields anywhere else in a message), resulting in the extra fields we see.

With this in mind, we can layout the standard format for all Nest messages, thus defining Nest's transporter message protocol.

#### Nest Transporter Message Protocol

1. Request messages (coming from [**Nest requestors**](#nest-as-requestor)) are wrapped in a structure that we can depict as follows:

   ```typescript
   {
     /**
      *  The message topic, also known inside Nest as
      *  the "message pattern"
      */
     pattern: string,
     /**
      *  The message payload (the first argument of
      *  ClientProxy#send or ClientProxy#emit)
      */
     data: any,
     /**
      *  A unique identifier, assigned by Nest.  Present
      *  only when the message is a request (created with
      *  ClientProxy#send)
      */
     id: string
   }
   ```

2. Response messages (coming from [**Nest responders**](#nest-as-responder)) are wrapped in a structure we can depict as follows:

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

Clearly, we'll have a problem communicating between our Nest and non-Nest apps based on these different message formats. For example, in the [**Nest as responder**](#nest-as-responder) case, we have the following issue, where an external request is not understood by the Nest responder due to the message format incompatibility.

![message format mismatch](./assets/mismatch.png 'Message Format Mismatch')
<a name="figure4"></a><figcaption>Figure 4: Message Format Mismatch</figcaption>

As you can imagine, we have the reverse issue in the [**Nest as requestor**](#nest-as-requestor) case, where Nest issues requests wrapped in the Nest request format, which aren't understood by the external app, and the external app also responds with an incompatible message format (missing fields expected by Nest).

**Now the big question**: How do we reconcile these message format differences to connect Nest and external NATS apps, as in <a href="https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3#figure1">Figure 1 Cases B, C and D</a>?

The good news is that **Nest anticipates this need and provides a neat solution.** We now have all the pieces in place to start seeing how Nest solves this problem and how to craft a solution. We'll dive into this in Part 3!

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
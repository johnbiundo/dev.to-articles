---
published: False
title: "Part 3: Completing the Server Component"
description: "tags: nestjs, nest, faye, microservices, node.js"
series: "Advanced NestJS Microservices"
cover_image: "https://dev-to-uploads.s3.amazonaws.com/i/lgt7m4a8dton9rvuimje.gif"
canonical_url:
---

*John is a member of the NestJS core team*

### Introduction

This is Part 3 of a six-part series.  If you landed here from Google, you may want to start with [Part 1](https://dev.to/nestjs/build-a-custom-transporter-for-nestjs-microservices-dc6-temp-slug-6007254?preview=d3e07087758ff2aac037f47fb67ad5b465f45272f9c5c9385037816b139cf1ed089616c711ed6452184f7fb913aed70028a73e14fef8e3e41ee7c3fc#requestresponse).

In this article, we build the final iteration of the server component of our Faye Custom Transporter.

**Reminder**: Many of the concepts and terminology here are introduced and explained in [this article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3). That series serves as a good foundation for understanding the more advanced concepts covered in this series.

#### Get the Code

All of the code in these articles is available [here](https://github.com/johnbiundo/nestjs-faye-transporter-sample).  As always, these tutorials work best if you follow along with the code.  The [README](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md) covers all the steps you need to get the repository, build the apps, and follow along.  It's easy!  I strongly encourage you to do so.  Note that each article has a corresponding branch in the repository.  For example, this article (Part 3), has a corresponding branch called `part3`. Read on, or [get more details here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-3-completing-the-server-component) on using the repository.

#### Git checkout the current version

For this article, you should `git checkout` the branch `part3`.  You can get more information [about the git repository branches here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-3-completing-the-server-component).

#### Build the Apps for This Part

For each article in the series, we introduce some new components (and sometimes entirely new **projects**).  For convenience, at the start of each article, you should run the command:\*

```bash
$ # from root directory of project (e.g., transporter-tutorial, or whatever you chose as the root)
$ sh build.sh
```
\**This is a shell script that runs on Linux-like systems.  You may have to make an adjustment for non-Linux systems.  Note that the script is simply a convenience that runs `npm install` inside each top-level directory, so you can always fall back to that technique if you have trouble with the script.*

### Overview

I ended the last article with a discussion of the shortcomings of our *Take 1* (first iteration) implementation of the Faye Custom Transporter server component.  In this article, we'll address those shortcomings. Most of this work is fairly straightforward, but I want to take a moment to discuss the "Observable issue" a bit further.

You **could actually ignore** this issue entirely.  The server component we built in the last chapter is functional, and we'll clean up a few loose ends here to make it typesafe, etc. However, by ignoring proper handling of Observables, you'd be leaving out a feature that you may not miss until you **really** need it.  Plus, let me hit you with the full commercial:
- It's super fun to understand how it works, and if you're not an [RxJS](https://rxjs-dev.firebaseapp.com/) whiz, don't worry, that won't stop you (**and** you might even pick up a few nifty tricks about RxJS along the way)
- It builds a deeper appreciation and understanding of the elegance of the NestJS framework
- It's really, really easy to implement, once you get on the happy path
- The feature provides some amazing benefits, virtually for free
- Your transporter really **should** be plug-and-play compatible with other transporters to be a good Nest citizen (Nestizen? :smiley:)

I could go on and on, but instead, I've encapsulated all the [concrete examples and detailed analysis here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/observable-deepdive.md) to try to keep this tutorial on track.  Suffice to say, I strongly encourage you to follow that link and take that little side trip to both understand why this is an important step &#8212; and because you might even get inspired to use Nest microservices in some new and interesting ways.

With all that said, let's jump in!

### Addressing Observables

After that build-up, you might think we're about to embark on some form of brain surgery (maybe I should say ":rocket: science" in keeping with the trekkie theme? :wink:).  Fortunately, that's not really the case.  Let's start by considering the requirements.

At the highest level, we need to consider the possibility that the user-land handler returns a stream (Observable that emits multiple values). Our earlier version won't handle that (as we saw at the end of the last article).  The problem lies with our response message publishing logic being unaware of a response **stream**.  It simply awaits a single response and publishes.  Let's take a quick look at the part of the code that publishes to refresh ourselves:

```typescript
// nestjs-faye-transporter/src/responder/transporters/server-faye.ts
// on branch part2
public getMessageHandler(pattern: string, handler: Function): Function {
  return async message => {
    const inboundPacket = this.deserializer.deserialize(message);
    const response = await handler(inboundPacket.data);
    const outboundRawPacket = {
      err: null,
      response,
      isDisposed: true,
      id: (message as any).id,
    };
    const outboundPacket = this.serializer.serialize(outboundRawPacket);
    this.fayeClient.publish(`${pattern}_res`, outboundPacket);
  };
}
```
We could try to handle this streaming ourselves, but it's a tad tricky bit of code, and as such, a great candidate for the framework to handle for us. Which it does! As always with frameworks, if we want to delegate a responsibility, we have to invert our thinking a bit. Rather than publishing directly, we instead need to **build** a publish function and hand it to the framework


Now, here's the new, improved, *Observable-aware* version. This is what you'll see now (on the current branch, `part3`) when you open the `nestjs-faye-transporter/src/responder/transporters/server-faye.ts` file:

```typescript
//nestjs-faye-transporter/src/responder/transporters/server-faye.ts
// on branch part3
public getMessageHandler(pattern: string, handler: Function): Function {
  return async (message: ReadPacket) => {
    const inboundPacket = this.deserializer.deserialize(message);
    const fayeCtx = new FayeContext([pattern]);

    const response$ = this.transformToObservable(
      await handler(inboundPacket.data, fayeCtx),
    ) as Observable<any>;

    const publish = (response: any) => {
      Object.assign(response, { id: (message as any).id });
      const outgoingResponse = this.serializer.serialize(response);
      return this.fayeClient.publish(`${pattern}_res`, outgoingResponse);
    };

    response$ && this.send(response$, publish);
  };
}
```

Let's discuss the differences.  For now, ignore the `fayeCtx` object &#8212; we'll cover that [below](#subscription-context).

The reorganized code does two main things:
1. It converts our handler response to an Observable
2. It builds a publish function that it can delegate to the framework (which will then use it to publish a stream if that's what the user-land handler returns)

For the first item, we call a built-in (inherited from `Server`) utility to convert the response to an Observable.

```typescript
const response$ = this.transformToObservable(
  await handler(inboundPacket.data, {}),
) as Observable<any>;
```

This step is pretty intuitive, and the utility is correspondingly pretty simple.  Feel free to take a moment to review the [`transformToObservable()` method's source code](https://github.com/nestjs/nest/blob/master/packages/microservices/server/server.ts#L108).

Building the publish function is also fairly straightforward.

```typescript
const publish = (response: any) => {
  Object.assign(response, { id: (message as any).id });
  const outgoingResponse = this.serializer.serialize(response);
  return this.fayeClient.publish(`${pattern}_res`, outgoingResponse);
};
```

This code is very similar to the way we published in the *Take 1* version (from branch `part2`), but now packaged as a function we can hand off to the framework.  Study it for a moment and make sure you understand it &#8212; it should be pretty straightforward. The `Object.assign(...)` code's job is to copy the `id` field from the inbound request to the outbound response.  We did that in the earlier version as well, but there we were defining the response object in place, so the technique was a little different.

**Note**: We still haven't discussed **why** we need that `id`.  For now, just trust the process :smiley:.  Don't worry, we'll cover this in [Part 5](https://dev.to/nestjs/part-5-completing-the-client-component-hlh-temp-slug-2907984?preview=82c11163db963ca01d8d62d3a7b14843b422a6b28f46762d999bbe4b7035ad634d48bbbdd740e36376121aa673354ff5259f8b3028bceb931e800d9e).

Finally, we delegate the publishing step to the framework with the final line:

```typescript
response$ && this.send(response$, publish);
```

The `send()` method is inherited from `Server`; it **takes care of the details of dealing with streams**.

> The significance of the previous sentence should not be overlooked.  To fully appreciate both **why** this is so useful and how the framework makes this as easy as delegating a `publish()` function, you really should [read this deep dive](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/observable-deepdive.md).

Anyway, let's discuss what the framework is doing for us.  As mentioned, `send()` is inherited from `Server`.  See [here](https://github.com/nestjs/nest/blob/master/packages/microservices/server/server.ts) for the full source code of the `Server`.  Let's reproduce it below to get a feel for it.

> Note: we **don't need to fully understand** how this method works, and I won't belabor it here.  It's fair to treat it as a black box, as long as we understand **how to use it**.  We'll make sure we do!

```typescript
// from https://github.com/nestjs/nest/blob/master/packages/microservices/server/server.ts
  public send(
    stream$: Observable<any>,
    respond: (data: WritePacket) => void,
  ): Subscription {
    let dataBuffer: WritePacket[] = null;
    const scheduleOnNextTick = (data: WritePacket) => {
      if (!dataBuffer) {
        dataBuffer = [data];
        process.nextTick(() => {
          dataBuffer.forEach(buffer => respond(buffer));
          dataBuffer = null;
        });
      } else if (!data.isDisposed) {
        dataBuffer = dataBuffer.concat(data);
      } else {
        dataBuffer[dataBuffer.length - 1].isDisposed = data.isDisposed;
      }
    };
    return stream$
      .pipe(
        catchError((err: any) => {
          scheduleOnNextTick({ err, response: null });
          return empty;
        }),
        finalize(() => scheduleOnNextTick({ isDisposed: true })),
      )
      .subscribe((response: any) =>
        scheduleOnNextTick({ err: null, response }),
      );
  }
  ```

While at first a little obscure, this really isn't **that** hard to understand if you work through it, though it does require some understanding of observables.  Essentially, we are subscribing to the response coming back from our user-land handler. If the subscription produces multiple values (a stream), we buffer them (`dataBuffer`), then publish each value using the `publish()` function we built a moment ago (inside this method the `publish()` function is accessed as the `respond` parameter). When the stream completes, we signify that by setting `isDisposed` to true on the final emitted response.

### Handle Events

Thus far we've skipped handling events &#8212; those user-written handlers that are decorated with `@EventPattern(...)`.  Let's take care of those now.  They turn out to be a lot easier than *request/response* type messages precisely because they don't require any response.  Take a look at the `bindHandlers()` method of `server-faye.ts`. The comments should pretty much explain the simple change we've made here to handle events.

```typescript
// nestjs-faye-transporter/src/responder/transporters/server-faye.ts
public bindHandlers() {
  /**
   * messageHandlers is populated by the Framework (on the `Server` superclass)
   *
   * It's a map of `pattern` -> `handler` key/value pairs
   * `handler` is the handler function in the user's controller class, decorated
   * by `@MessageHandler()` or `@EventHandler`, along with an additional boolean
   * property indicating its Nest pattern type: event or message (i.e.,
   * request/response)
   */
  this.messageHandlers.forEach((handler, pattern) => {
    // In this version (`part3`) we add the handler for events
    if (handler.isEventHandler) {
      // The only thing we need to do in the Faye subscription callback for
      // an event, since it doesn't return any data to the caller, is read
      // and decode the request, pass the inbound payload to the user-land
      // handler, and await its completion.  There's no response handling,
      // hence we don't need all the complexity of `getMessageHandler()`
      this.fayeClient.subscribe(pattern, async (rawPacket: ReadPacket) => {
        const fayeCtx = new FayeContext([pattern]);
        const packet = this.parsePacket(rawPacket);
        const message = this.deserializer.deserialize(packet, {
          channel: pattern,
        });
        await handler(message.data, fayeCtx);
      });
    } else {
      this.fayeClient.subscribe(
        `${pattern}_ack`,
        this.getMessageHandler(pattern, handler),
      );
    }
  });
}
```
Two additional (unrelated) changes that show up on this branch are:

1. The use of the `FayeContext` object. We'll cover that in the next section.
2. Passing `{ channel: pattern }` to the `deserialize()` call.  We'll cover that [later in the article](#deserialization-options).

### Subscription Context

Consider what happens when we subscribe to a topic via the broker API.  With Faye, it's straightforward.  We register a *Faye subscription handler* on a topic, and when a message matching that topic arrives, our handler is called with the message payload containing **only what the publisher sent**.

For some brokers, the subscription handler can receive additional context about the message &#8212; sometimes in the callback parameters, and sometimes in the message payload.  For example, with NATS, the message payload packs the actual content sent by the publisher in a top-level `data` property, and adds two additional top-level properties with context metadata: one describing the topic the caller was subscribed to, and one containing the optional `reply` topic. We refer to this metadata as `Context`, and Nest allows you to pass this information through to the user-land handler (method decorated by `@MessagePattern()` or `@EventPattern()`).

Since Faye simply doesn't provide any such metadata, we'll demonstrate this behavior by passing the *channel* in this context object.  This is not particularly useful in this case, as we're not actually providing any **new** information, but with Faye it's the best we can do to demonstrate the concept of the `Context` object.

#### Creating the context object

For any given broker, the technique for creating the context object may vary, based on the discussion above. In the end, you'll need to extract any context information (from the message payload, if context is encoded there, or from the `subscribe()` callback parameters) and pass it to the user-land handlers. We use a custom object to pass context.  Open the file `nestjs-faye-transporter/src/responder/ctx-host/faye-context.ts`.  It's a pretty simple object that extends `BaseRpcContext`.

```typescript
// nestjs-faye-transporter/src/responder/ctx-host/faye-context.ts
import { BaseRpcContext } from '@nestjs/microservices/ctx-host/base-rpc.context';

type FayeContextArgs = [string];

export class FayeContext extends BaseRpcContext<FayeContextArgs> {
  constructor(args: FayeContextArgs) {
    super(args);
  }

  /**
   * Returns the name of the Faye channel.
   */
  getChannel() {
    return this.args[0];
  }
}
```

Essentially, it keeps an array of context values &#8212; in our case just strings, though you can specify a more complex type if you need to.  For example, [MQTT](https://github.com/nestjs/nest/blob/master/packages/microservices/ctx-host/mqtt.context.ts#L3) uses:

```typescript
// from https://github.com/nestjs/nest/blob/master/packages/microservices/ctx-host/mqtt.context.ts
type MqttContextArgs = [string, Record<string, any>];

export class MqttContext extends BaseRpcContext<MqttContextArgs> {
  constructor(args: MqttContextArgs) {
    super(args);
  }
```

For each context value type (e.g., "channel", in our case, used for carrying the channel/pattern name), you supply a getter function.  Remember, this is a simple data structure as there are generally very few context values, so your getter is just pulling out a particular row of the array.

#### Passing Context to User-land Handlers

With this in place, we then need to simply find the right places to:
1. Instantiate a `FayeContext` object with the correct values
2. Pass it to the user-land handlers

We already encountered this behavior in the previous sections. Let's look again at the `getMessageHandler()` method, focusing on the relevant snippet from that method:

```typescript
  return async (message: ReadPacket) => {
    const inboundPacket = this.deserializer.deserialize(message);
    const fayeCtx = new FayeContext([pattern]);

    const response$ = this.transformToObservable(
      await handler(inboundPacket.data, fayeCtx),
    ) as Observable<any>;
```

It should be clear how we're using an instance of `FayeContext` here.  We instantiate it with the relevant context, and then pass the instance to our user-land handler method.  The code above takes care of context for *request/response* type handlers. We also need to handle this appropriately for *events*. We saw where that happened in the last section, in the `bindHandlers()` method.  Here's the relevant snippet:

```typescript
    if (handler.isEventHandler) {
      // The only thing we need to do in the Faye subscription callback for
      // an event, since it doesn't return any data to the caller, is read
      // and decode the request, pass the inbound payload to the user-land
      // handler, and await its completion.  There's no response handling,
      // hence we don't need all the complexity of `getMessageHandler()`
      this.fayeClient.subscribe(pattern, async (rawPacket: ReadPacket) => {
        const fayeCtx = new FayeContext([pattern]);
        const packet = this.parsePacket(rawPacket);
        const message = this.deserializer.deserialize(packet);
        await handler(message.data, fayeCtx);
      });
```
Remember, where and how you gather the context for any particular broker may vary, but the concept remains the same.

#### Testing Context

In this branch (`part3`), we provided a new project, `nestHttpApp`.   This is a simple Nest HTTP app that includes a few routes for testing our `nestMicroservice` over the Faye transporter. It's worth mentioning that this branch also includes a **full final version** of the `ClientFaye` subclass of `ClientProxy` &#8212; this is necessary for our `nestHttpApp` to instantiate a client to be able to run `ClientProxy#send()` and `ClientProxy#emit()` calls.  We'll actually build that `ClientFaye` component from scratch in the next two articles, but it's sitting there in this branch to help you test.  With this in place, you can see the context by following these steps:

1. Start up the `nestHttpApp`

    If you ran the `build.sh` script as described at the beginning of this chapter, all of this project's dependencies have been installed.  If not, simply open a terminal in the top level `nestHttpApp` directory and run `npm install`.  Then, start the application with `npm run start:dev`.
2. Issue an HTTP request like:

    > GET /customers

    **Note:** You can do this in any number of ways &#8212; via a browser, browsing to `localhost:3000/customers`, via something like [HTTPie - my favorite](https://httpie.org/), or with a desktop program like [Postman](https://www.postman.com/). See [here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-3-completing-the-server-component) for more help on running these requests.

Keep an eye on the `nestMicroservice` app's console log.  You should see a line like the following, demonstrating that we've successfully passed the ` FayeContext` data to the handler:
> [AppController] Faye Context: {"args":["/get-customers"]}"

### Deserialization Options

The transporter main class we inherit from (`Server`) has hooks for calling custom serialization and deserialization methods to encode/decode broker messages. I cover this topic extensively [in this article](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-3-4m20).  The main use case is for translating between non-Nest (i.e., external) message formats and the internal [Nest microservice transporter message protocol](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-2-3hgd). One thing we need to do to fully enable custom deserialization (i.e., let users hook their own custom deserializer class) with our custom transporter is fully implement the call to the `Deserializer#deserialize()` method.  Here's its interface:

```typescript
export interface Deserializer<TInput = any, TOutput = any> {
  deserialize(value: TInput, options?: Record<string, any>): TOutput;
}
```

This means we can pass in some context via a second argument that the user can access as the `options` parameter to utilize in their custom deserialization code.  Usually this context is just the *channel* (pattern), but it can also include other contextual information.  For example, the [NATS transporter](https://github.com/nestjs/nest/blob/master/packages/microservices/server/server-nats.ts#L94) passes the following information (note the `replyTo` property):

```typescript
const message = this.deserializer.deserialize(rawMessage, { channel, replyTo });
```

In our case, with Faye, we'll stick to the usual pattern and pass the channel.  This is done in `bindHandlers()`, where we added the deserialization context as shown below:

```typescript
    const message = this.deserializer.deserialize(packet, {
      channel: pattern,
    });
```

With this in place, any user-written deserializer will have access to the channel name to make decisions about transforming inbound messages. I encourage you to read this [article](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-3-4m20) for a deeper understanding of deserialization to help inform what you should pass as deserialization options.

### Type Safety

In this branch, we added types to several calls to help ensure type safety when end users use the custom transporter.  You can examine the code and explore how we use the following newly imported interfaces:

```typescript
// nestjs-faye-transporter/src/responder/transporters/server-faye.ts
import { FayeClient } from '../../external/faye-client.interface';
import { FayeOptions } from '../../interfaces/faye-options.interface';
```

Let's take a quick moment to examine the `FayeOptions` interface:

```typescript
// nestjs-faye-transporter/src/interfaces/faye-options.interface.ts
import { Serializer, Deserializer } from '@nestjs/microservices';

export interface FayeOptions {
  /**
   * faye server mount point (e.g., http://localhost:8000/faye)
   */
  url?: string;
  /**
   * time in seconds to wait before assuming server is dead and attempting reconnect
   */
  timeout?: number;
  /**
   * time in seconds before attempting a resend a message when network error detected
   */
  retry?: number;
  /**
   * connections to server routed via proxy
   */
  proxy?: string;
  /**
   * per-transport endpoint objects; e.g., endpoints: { sebsocket: 'http://ws.example.com./'}
   */
  endpoints?: any;
  /**
   * backoff scheduler: see https://faye.jcoglan.com/browser/dispatch.html
   */
  // tslint:disable-next-line: ban-types
  scheduler?: Function;
  /**
   * instance of a class implementing the serialize method
   */
  serializer?: Serializer;
  /**
   * instance of a class implementing the deserialize method
   */
  deserializer?: Deserializer;
}
```

These options (modulo `serializer` and `deserializer`, which are consumed by the `ServerFaye` class directly) are passed through to the Faye client library in `createFayeClient()` to customize the [Faye connection](https://faye.jcoglan.com/browser.html):

```typescript
  public createFayeClient(): FayeClient {
    // pull out url, and strip serializer and deserializer properties
    // from options so we conform to the `faye.Client()` interface
    const { url, serializer, deserializer, ...options } = this.options;
    return new faye.Client(url, options);
  }
```

The other benefit of this interface is that it provides intellisense for users constructing the `options` object in the call to `NestFactory.createMicroservice(...)` in the `main.ts` file.

![Editor Intellisense](./assets/faye-options-intellisense.png 'Editor Intellisense')
<figcaption><a name="intellisense"></a>Figure 1: Editor Intellisense</figcaption>

### Error Handling

What should we do if the server loses its connection to the broker (e.g., the Faye broker goes offline)?  This is a good question that probably deserves deeper discussion than what we'll cover here.  The Faye client library appears to be fairly [robust at re-establishing a connection](https://faye.jcoglan.com/browser.html), including [buffering failed attempts](https://faye.jcoglan.com/browser/dispatch.html), so we are going to simply defer to it, and log an error message to the console whenever a connection goes down.  To handle this, we simply add a listener for a [disconnect event](https://faye.jcoglan.com/browser/transport.html) and log the message:

```typescript
  public handleError(stream: any) {
    stream.on(ERROR_EVENT, (err: any) => {
      this.logger.error('Faye Server offline!');
    });
  }
```

### Checkpoint

We covered a lot of ground in this chapter. Because we provided the `nestHttpApp` and a full implementation of the Faye flavored `ClientProxy`, you now have a **fully working Faye Custom Transporter**.  You can feel free to experiment at large with the `nestHttpApp` and `nestMicroservice`.  Try issuing HTTP requests to the `nestHttpApp` like:

> GET /customers

and

> POST /customer  // <-- with a payload

See [here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-3-completing-the-server-component) for more help on running these and similar tests..

As mentioned earlier in this article, there's also a series of routes related to exploring how microservices handle promises and observables when they are returned from a **request**.  You can [read more here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/observable-deepdive.md).

### What's Next

In [Part 4](https://dev.to/nestjs/part-4-basic-client-component-298b-temp-slug-9977921?preview=21ec3d333fc6d9d92c11dcbd8430a5132e93390de84cb4804914aa143492e925e4299ca3eb7f376918c1ed77df56e29db2572e5d6f7ab235b3e5f2b9), we turn our attention to the "client" side of the transporter.  We'll build the Faye-flavored `ClientProxy` from scratch.  Like the server construction we've done, we'll break this into two parts so we can work through the main concepts first, then handle some edge cases and details in the subsequent part.

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
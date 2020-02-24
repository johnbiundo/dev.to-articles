---
published: False
title: "Part 3: Completing the Server Component"
description: "tags: nestjs, nest, faye, microservices, node.js"
series: "Advanced NestJS Microservices"
cover_image: "https://dev-to-uploads.s3.amazonaws.com/i/0pg4mzdjilg9qfpn819w.gif"
canonical_url:
---

*John is a member of the NestJS core team*

xxx longdash:
&#8212;

### Introduction

#### Background

Many of the concepts and terminology here are introduced and explained in [this article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3). That series serves as a good foundation for understanding the more advanced concepts covered in this series.

#### Get the Code

All of the code in these articles is available [here]().  As always, these tutorials work best if you follow along with the code.  The [README]() covers all the steps you need to get the repository, build the apps, and follow along.  It's easy!  I strongly encourage you to do so.

#### Git checkout the current version

**part4**

#### Build the Apps for This Part

### Overview

I ended the last article with a discussion of the shortcomings of our "Take 1" (first iteration) implementation of the Faye Custom Transporter server component.  In this article, we'll address those shortcomings. Most of this work is fairly straightforward, but I want to take a moment to discuss the "Observable issue" a bit further.

You **could** actually ignore this issue entirely.  The server component we built in the last chapter is functional, and we'll clean up a few loose ends here to make it typesafe, etc.  However, by ignoring proper handling of Observables, you'd be leaving out a feature that you may not miss until you **really** need it.  Plus, let me hit you with the full commercial:
- It's super fun to understand how it works, and if you're not an `RxJS` whiz, that won't stop you (**and** you might even pick up a few nifty tricks about `RxJS` along the way)
- It builds a deeper appreciation and understanding of the elegance of the Nest framework
- It's really, really easy to implement, once you get on the right path
- The feature provides some amazing benefits, virtually for free
- Your transporter really **should** be plug-and-play compatible with other transporters to be a good Nest citizen (Nestizen? :smiley:)

I could go on and on! And in fact, I do sort of go on, but I've decided to encapsulate all the [concrete examples and detailed justification here]() to try to keep this tutorial on track.  Suffice to say, I strongly encourage you to read that section to both understand why this is an important step, and to possibly get inspired to use Nest microservices in some new and interesting ways.

With all that said, let's jump in!

### Addressing Observables

After that build-up, you might think we're about to do some sort of brain surgery.  Fortunately, that's not really the case. The magic is all enabled by the framework, and it happens in our `getMessageHandler()` method. Let's quickly take a look at the one we built in the last article. If it's been a while, it's worth [re-reading that section of the last article]() to refresh yourself.  Here's the code:

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

Now, here's the *Observable-aware* version. This is what you'll see now (on the current branch, `part3`) when you open the `nestjs-faye-transporter/src/responder/transporters/server-faye.ts` file:

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

Let's discuss the differences.  For now, ignore the `fayeCtx` object &#8212; we'll cover that [below](#subscription-context).  At the highest level, we're converting our handler response (`handler(inboundPacket.data, {})` &#8212; which is the result we get back from the user's handler function) to an observable and **publishing** that observable (recall that *publishing* is how we send the response back to the requestor through the Faye broker). Let's walk through the code at the next level of detail:
1. We run a built-in (inherited from `Server`) utility to convert the response to an observable.
    ```typescript
    const response$ = this.transformToObservable(
      await handler(inboundPacket.data, {}),
    ) as Observable<any>;
    ```

    This step is pretty intuitive, and the utility is correspondingly pretty simple.  Take a moment to review the [source code](https://github.com/nestjs/nest/blob/master/packages/microservices/server/server.ts#L108).

2. The more magical part is in the publish step.  Let's discuss that in some detail.

To publish the observable, we break the publish step down into two sub-steps:
1. Rather than invoke `fayeClient.publish()` directly, as in *Take 1*, we instead *build* a `publish()` function.  We're going to delegate publishing to the framework, so passing it this built function is the way we'll do it.  We build the publish function is these lines:
    ```typescript
    const publish = (response: any) => {
      Object.assign(response, { id: (message as any).id });
      const outgoingResponse = this.serializer.serialize(response);
      return this.fayeClient.publish(`${pattern}_res`, outgoingResponse);
    };
    ```

    This code is very similar to the way we published in the *Take 1* version (from branch `part2`).  Study it for a moment and make sure you understand it &#8212; it should be pretty straightforward. The `Object.assign(...)` code's job is to copy the `id` field from the inbound request to the outbound response.  (And no, we still haven't discussed **why** we need that `id`.  For now, just trust the process :smiley:.  Don't worry, we'll cover this in [Part 4]()).
2. Finally, we have the somewhat mysterious looking `response$ && this.send(response$, publish);`. What's up with that?  This is the step that actually **performs** the publishing of the message.  The `send()` method is provided by the framework, and it **takes care of the details of dealing with Observables**.

> Once again, the importance of the previous sentence can not be overstated.  To fully appreciate both **why** this is so useful and how the framework makes this as easy as delegating a `publish()` function, you really should [read this deep dive]().

Anyway, let's discuss what's happening with this step.  The `send()` method is inherited from `Server`.  See [here]() for the full source code of the `Server`.  Let's reproduce it below to get a feel for it.

> Note: we don't **need** to fully understand how this method works, and I won't belabor it here.  It's fair to treat it as a black box, as long as we understand **how** to use it.  We'll make sure we do!

```typescript
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
This little bit of magic, at 30 lines of code, enables us to **stream** a remote observable to our calling `ClientProxy#send()` call.  Again, read more about [what that means and why you probably do care here](). When (I didn't say "if" :smiley:) you do read that section, I promise you some lightbulbs will go off.  Among other things, if you've been wondering about that `isDisposed` property of a response message, that will become clear.  This is a wonderful example of something that once you explore it, you'll feel a lot more comfortable with the *"casual magic"* you sometimes encounter with Nest, and begin to appreciate that as we fill in the gaps in the documentation, that magic is really just the manifestation of a truly elegant architecture.

#### What's the Recipe?

Let's go from the specific case above to the recipe you should use when building any arbitrary transporter.

1. Implement the code that handles responding to requests (that is, code that **typically** runs a broker client API `subscribe()`-like command responsible for registering the response handler with the broker).  In our example, this is done with the `bindHandlers()` method.  In this method, you **must dynamically construct** the call you are registering with `subscribe()`.  That's the job of the next step.
2. Call a method to dynamically construct the call to the response handler being registered.  In our example, this is done with the `getMessageHandler()` method.  That method is responsible for the following steps:

    1. Deserialize the inbound request
    2. Build an observable response using the built-in inherited `this.transformToObservable()` utility, passing in an *awaited* call to the *target handler* (the *target handler* is the user-supplied method bound to a pattern with `@MessagePattern()`).
    3. Build a `publish()` function.  That function should always take a single input argument, and return a function call that runs the native `publish()` call (or the broker client API equivalent).
    4. Invoke the built-in inherited `this.send()` helper method, passing in the observable constructed in step  and the publish function built in step 3.

An implementation for another broker will probably look something like the Faye strategy above, but the details can vary because the protocols and APIs for different brokers vary.  In [Part 6]() of this series, we'll take a look at a few different broker implementations to see how they vary.

### Handle Events

Thus far we've skipped handling events &#8212; those user-written handlers that are decorated with `@EventPattern(...)`.  Let's take care of those now.  They turn out to be a lot easier than *request/response* type messages precisely because they don't require any response.  Take a look at the `bindHandlers()` method of `server-faye.ts`. The comments should pretty much explain the simple change we've made here.  Two additional changes are:

1. The use of the `FayeContext` object. We'll cover that in the next section.
2. Passing { channel: pattern } to the `deserialize()` call.  We'll cover that [later in the article](#deserialization-options).

```typescript
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

### Subscription Context

Consider what happens when we subscribe to a topic via the broker API.  With Faye, it's straightforward.  We register a callback handler on a topic, and when a message matching that topic arrives, our handler is called with the message payload containing **only what the publisher sent**.

For some transporters, the subscription handler can receive additional context about the message, sometimes in the callback parameters, and sometimes in the message payload.  For example, with NATS, the message payload packs the actual content sent by the publisher in the `data` property, as well as two context metadata fields: one describing the topic the caller was subscribed to, and one containing the optional `reply` topic. We refer to this metadata as `Context`, and Nest allows you to pass this information through to the user-land handler (method decorated by `@MessagePattern()` or `@EventPattern()`).

Since Faye doesn't have such metadata, we'll demonstrate this behavior by simply passing the *channel* in this context object.  This is not necessary, as it doesn't add any new information, but it does demonstrate the concept of the `Context` object.

#### Creating the context object

For any given broker, the technique may vary, based on the discussion above. In the end, you'll need to extract any context information (from the message payload, if context is encoded there, or from the callback parameters) and pass it to the user-land handlers. We use a custom object to pass context.  Open the file `nestjs-faye-transporter/src/responder/ctx-host`.  It's a pretty simple object that extends `BaseRpcContext`.

```typescript
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

Essentially, it keeps an array of context values &#8212; in our case just strings, though you can specify a more complext type if you need to.  For example, [MQTT](xxx) uses:

```typescript
type MqttContextArgs = [string, Record<string, any>];

export class MqttContext extends BaseRpcContext<MqttContextArgs> {
  constructor(args: MqttContextArgs) {
    super(args);
  }
```

For each context value type (e.g., "channel" in our case, for carrying the channel/pattern name), you supply a getter function.  Remember, this is a simple data structure as there are generally very few contexc values, so your getter is just pulling out a particular row of the array.

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

It should be clear how we're using an instance of `FayeContext` here.  We instantiate it with the relevant context, and then pass the instance to our user-land handler method.  This takes care of **request/response** type handlers. We also need to handle this appropriately for events. We saw that in the last section, in the `bindHandlers()` method.  Here's the relevant snippet:

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

In this branch (`part3`), we provided a new project, `nestHttpApp`, to exercise the context feature.  We'll focus a lot more on the client side, and use `nestHttpApp` extensively, in future chapters.  For now, you can see the context by following these steps:

1. Start up the `nestHttpApp`
    If you ran the `build.sh` script as described at the beginning of this chapter, all of this project's dependencies have been installed.  If not, simply open a terminal in the top level `nestHttpApp` directory and run `npm install`.  Then, start the application with `npm run start:dev`.
2. Issue a request like:

    > GET /customers

    **Note:** You can do this in any number of ways -- via a browser, browsing to `localhost:3000/customers`, via something like [HttPie - my favorite](https://httpie.org/), or something like [Postman](https://www.postman.com/).

Keep an eye on the `nestMicroservice` app's console log.  You should a line like:
> [AppController] Faye Context: {"args":["/get-customers"]}"

### Deserialization Options

The transporter main (`Server`) class has hooks for serialization and deserialization of broker messages. I cover this topic extensively [in this article](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-3-4m20).  The main use case is for translating between non-Nest (i.e., external) message formats and the internal Nest microservice transporter message protocol. One thing we need to do to fully enable custom deserialization (i.e., let users hook their own custom deserializer class) with our custom transporter is fully implement the call to the `Deserializer#deserialize`.  Here's the interface:

```typescript
export interface Deserializer<TInput = any, TOutput = any> {
  deserialize(value: TInput, options?: Record<string, any>): TOutput;
}
```

This means we can pass in some context that the user can utilize in their custom deserialization code.  Usually this is the *channel* (pattern), but can also include other contextual information.  For example, the [NATS transporter](https://github.com/nestjs/nest/blob/master/packages/microservices/server/server-nats.ts#L94) passes the following information:

```typescript
const message = this.deserializer.deserialize(rawMessage, { channel, replyTo });
```

In our case, with Faye, we'll stick to the usual pattern and pass the channel.  This is done in `bindHandlers()`, where we added the deserialization context as shown below:

```typescript
    const message = this.deserializer.deserialize(packet, {
      channel: pattern,
    });
```

With this in place, any user written deserializer will have access to the channel name to make decisions about transforming inbound messages. I encourage you to read this [article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-3-4m20) for a deeper understanding of deserialization to help inform what you should pass as deserialization options.

### Type Safety



### Error Handling

### Checkpoint

### What's Next

In [Part 4](xxx), we cover:
* A
* B
* C

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
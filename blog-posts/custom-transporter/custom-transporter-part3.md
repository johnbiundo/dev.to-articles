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

    const response$ = this.transformToObservable(
      await handler(inboundPacket.data, {}),
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

Let's discuss the differences.  At the highest level, we're converting our handler response (`handler(inboundPacket.data, {})` &#8212; which is the result we get back from the user's handler function) to an observable and **publishing** that observable (recall that *publishing* is how we send the response back to the requestor through the Faye broker). Let's walk through the code at the next level of detail:
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

> nce again, the importance of the previous sentence can not be overstated.  To fully appreciate both **why** this is so useful and how the framework makes this as easy as delegating a `publish()` function, you really should [read this deep dive]().

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

Thus far we've skipped handling events &#8212; those user-written handlers that are decorated with `@EventPattern(...)`.  Let's take care of those now.  They turn out to be a lot easier than *request/response* type messages precisely because they don't require any response.

### What's Next

In [Part 4](xxx), we cover:
* A
* B
* C

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
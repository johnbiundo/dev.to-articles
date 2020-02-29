---
published: False
title: "Part 5: Completing the Client Component"
description: "tags: nestjs, nest, faye, microservices, node.js"
series: "Advanced NestJS Microservices"
cover_image: "https://dev-to-uploads.s3.amazonaws.com/i/lgt7m4a8dton9rvuimje.gif"
canonical_url:
---

*John is a member of the NestJS core team*

### Introduction

This is part 5 of a six-part series.  If you landed here from Google, you may want to start with [part 1](https://dev.to/nestjs/build-a-custom-transporter-for-nestjs-microservices-dc6-temp-slug-6007254?preview=d3e07087758ff2aac037f47fb67ad5b465f45272f9c5c9385037816b139cf1ed089616c711ed6452184f7fb913aed70028a73e14fef8e3e41ee7c3fc#requestresponse).

In this article, we build the final iteration of the client component of the Faye Custom Transporter.

**Reminder**: Many of the concepts and terminology here are introduced and explained in [this article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3). That series serves as a good foundation for understanding the more advanced concepts covered in this series.

#### Get the Code

All of the code in these articles is available [here](https://github.com/johnbiundo/nestjs-faye-transporter-sample).  As always, these tutorials work best if you follow along with the code.  The [README](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md) covers all the steps you need to get the repository, build the apps, and follow along.  It's easy!  I strongly encourage you to do so.  Note that each article has a corresponding branch in the repository.  For example, this article (part 5), has a corresponding branch called `part5`. Read on, or [get more details here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-5-final-client-component) on using the repository.

#### Git checkout the current version

For this article, you should `git checkout` the branch `part4`.  You can get more information [about the git repository branches here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-5-final-client-component).

#### Build the Apps for This Part

For each article in the series, we introduce some new components (and sometimes entirely new **projects**).  For convenience, at the start of each article, you should run the command\*

```bash
$ # from root directory of project (e.g., transporter-tutorial, or whatever you chose as the root)
$ sh build.sh
```
\**This is a shell script that runs on Linux-like systems.  You may have to make an adjustment for non-Linux systems.  Note that the script is simply a convenience that runs `npm install` inside each top-level directory, so you can always fall back to that technique if you have trouble with the script.*

### Overview

I ended the last article with a discussion of the shortcomings of our *Take 1* (first iteration) implementation of the Faye Custom Transporter client component.  In this article, we'll address those shortcomings.

### Strategy for Solving the Multiplexing Challenge

In the last chapter, we encountered what I called the *multiplexing challenge*.  Let's analyze that problem further and begin to put together a strategy to solve it.  Buckle up, as this is definitely the most complex challenge we've faced yet.

> **Note**: It's helpful to come at this starting from a somewhat abstract level, and successively refine our understanding and implementation.  From a practical standpoint, while we will walk through the details of handling this in our Faye client component, some of the details will vary from one transport library (e.g., broker) to the next because **each has its own message pattern and API**.  If you understand the concepts, there is a certain amount of boilerplate material that you can more-or-less adapt to a particular transport library.  Bottom line: pay more attention to the concepts than to understanding every line of code.  Inevitably, if and when you attempt to build a new transporter client, you'll have to work through a few knotholes.  Until then, focus more on the **strategy** than on every single tactic and line of code.

OK, let's start working on the strategy.  We're going to work through a number of components of the strategy starting at a somewhat abstract level.

#### The Observable Part

We know `ClientProxy#send()` returns an Observable.  The job of that Observable is to consume incoming Faye messages (boxes in the diagram below) and emit them as a stream (circles in the diagram below) so that our user-land subscriber can deal with them as a nice well-behaved RxJS observable.

![Convert Messages to Stream](./assets/client-proxy-observable.gif 'Convert Messages to Stream')
<figcaption><a name="remote-stream"></a>Figure 1: Convert Messages to Stream</figcaption>

Everything we do from here on in is going to be packaged in an Observable that we return to the client. "But John", I hear you say, "we already did that in the last chapter!"  Alas, as we discovered at the end of the chapter, we have to account for more complexity.  Basically, we have to account for the fact that since our `ClientProxy` is operating in the context of a shared server (e.g., our `nestHttpApp`) it will actually be dealing with **multiple** observables simultaneously --xxx-- each representing a unique outstanding `ClientProxy#send` request.

One valuable lesson we learned in the last chapter is that we can use the Observable/observer pattern with a factory method to "wrap our Faye client inside an Observable". This was our *observable subscription function factory* (the `handleRequest()` method which builds the subscription function --xxx-- the bit of code that actually emits the value through our Observable).  We're going to build on this lesson, and take it to the next level, to deal with the complexities of the multiplexing challenge.

#### Handling Multiple Requests: The Union of Observables and Correlation Ids

Let's break down the multiple outstanding requests challenge a bit further.  If we think about the Observable we return, it represents a future stream of responses to our requests.  We already return a unique observable for each `ClientProxy#send` invocation, but our *observable subscription function factory* is too naive.  It binds each observable to the same observable subscription function.  That single shared observable subscription function (seen below) sends a response **to each observer** whenever a message comes in on our response channel, which is why we failed our race condition test.

```typescript
// nestjs-faye-transporter/src/requestor/cleints/faye-client.ts
// from branch `part4`
...
    const subscriptionHandler = rawPacket => {
      const message = this.deserializer.deserialize(rawPacket);
      const { err, response, isDisposed } = message;
      if (err) {
        return observer.error(err);
      } else if (response !== undefined && isDisposed) {
        observer.next(response);
        return observer.complete();
      } else if (isDisposed) {
        return observer.complete();
      }
      observer.next(response);
    };
...
```

So we need a smarter strategy.  Such a strategy needs to do dynamic binding of the *observable subscription function* logic.  It needs to delay binding the results of the *observable subscription function factory* until a request is made. We have to make a little leap here.  Conceptually, what we're going to do is **store** an instance of that *observable subscription factory function* each time a request is made. Later, when we get a response, we'll retrieve that stored function and use it to give us a unique *observable subscription factory function* for the request.  For the rest of this article, we'll refer to this stored function as our *handler factory*.

This construction let's us associate a **unique** *observable subscription function* with each observable.  The approach we'll take goes something like this:

1. At the time a user-land request is issued (via `ClientProxy#send`), we'll create an instance of the *handler factory*, along with a unique identifier for the request. We'll store an association between the unique id and the *handler factory* in a map.
2. When a response is received, we'll extract the identifier from the response, and use it to look up the *handler factory*. We'll then use this unique instance of the *handler factory* to give us the actual final *observable subscription function* associated with the request, and use this to emit the result.  Since the only observable associated with this *observable subscription function* is the originator of the request, that's the only one that receives the emitted result.
3. To tie these things together, we need to pass the identifier all the way through the request/response flow.  The requesting code generates the identifier, it's attached (as the `id` property) the outbound message, and finally returned on the corresponding inbound message. Then we use it as described in step 2 above.

> The pattern we described in step 3 above is often referred to as using [correlation ids](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html) (for example, here's a [tutorial showing how](https://www.rabbitmq.com/tutorials/tutorial-six-python.html) a similar concept can be used for writing native RabbitMQ client apps).

#### Other Bookkeeping

Another element of our strategy is to manage Faye channel subscription intelligently.  As we know, multiple inbound requests to `nestHttpApp` results in multiple outbound requests to, and responses from, `nestMicroservice`.  We've been dealing with the multiplexing aspect of that challenge, but let's introduce one more requirement.  An additional consequence of our design goal of **sharing a single inbound response channel** is that we we **must have only a single active subscription** to that response channel.  When all active requests have completed, we unsubscribe.  When a new request comes in, we subscribe again, and leave the subscription open until the channel quiesces (i.e., there are no more inflight requests) again.

#### Connection Management

One thing we'll find, as we integrate our code with the framework, is that we need to adhere to its expectations for how we provide a client library connection.  We'll mostly rely on some boilerplate code here, but let's briefly describe the strategy. Since we're packaging up a bunch of stuff inside *observable subscription functions* (and their factories, etc.), the framework expects us to provide access to the connection in a particular way.  For the most part, we can just utilize some boilerplate to handle this.  There's only a small bit that is specific to a client library.  This is all packaged up in a `connect()` method that we must implement in our `ClientFaye` class.

#### Odds and Ends

We'll also see sprinkled in some functions that deal with other loose ends:
* we return channel names using a function, rather than hard coding them with the `_ack` and `_res` modifiers to avoid scattering magic strings throughout the code
* since user-land *patterns* can be arbitrary objects (we usually use strings, but Nest allows constructs like `client.send({cmd: 'get-customers'}, {})`), we use a built-in inherited function to "normalize" (essentially flatten) the pattern for internal usage

### Implementation

With this strategy in mind, here's the outline of how we'll implement it.

> Note that there's a fair amount of interaction back and forth between our `ClientFaye` subclass and the `ClientProxy` superclass it extends, so pay close attention to that, and I'll try to give clear signposts to guide you.

1. Unlike our *Take 1* version, where we created a new standalone class, we're going to extend the `ClientProxy` class.  This is, of course, how we "plug in" to the framework.
2. We're going to let the framework take over the implementation of `send()` for us.  That's the entry point for processing a user-land `send()` call, and allows us to plug in our pieces without disrupting the overall machinery.
3. The superclass `send()` method calls upon a concrete implementation of `publish()`, which is where we'll implement our "factory of factories" and deal with unique identifiers and so forth, [as described above](#handling-multiple-requests-the-union-of-observables-and-correlation-ids).
4. We're going to implement a method to *unsubscribe* from a Faye topic.
5. We're going to provide a concrete implementation of `dispatchEvent()`, which is how `ClientProxy#emit()` is handled.
6. We're going to beef up *connection management*, as we mentioned above, to enable the framework to efficiently share our connection across multiple calls.
7. We'll take care of a few other minor details.

We have our shopping list, so let's get started!

### *Take 2* Code Review

#### The Superclass `send()` Method

Let's begin at a logical place, the `send()` method.  As mentioned, we're going to let the superclass handle this for us, and plug in our code appropriately.  Let's take a look:

```typescript
// From the ClientProxy superclass.  See source code here:
// https://github.com/nestjs/nest/blob/master/packages/microservices/client/client-proxy.ts#L82
  public send<TResult = any, TInput = any>(
    pattern: any,
    data: TInput,
  ): Observable<TResult> {
    if (isNil(pattern) || isNil(data)) {
      return _throw(new InvalidMessageException());
    }
    return defer(async () => this.connect()).pipe(
      mergeMap(
        () =>
          new Observable((observer: Observer<TResult>) => {
            const callback = this.createObserver(observer);
            return this.publish({ pattern, data }, callback);
          }),
      ),
    );
  }
```

Stripping the superclass `send()` method to the bone, here's what it's doing:
1. Reusing a `connection` if it exists or obtaining a new one.  Note that since it depends on client library particulars to deal with obtaining a connection, it depends on the `connect()` method --xxx-- which we are responsible for implementing and will get to soon.
2. Creating the Observable that is effectively the "wrapper" where we'll implement the strategy we [discussed above](#strategy-for-solving-the-multiplexing-challenge).

Let's look at that Observable creation step for a moment.  The first line `const callback = this.createObserver(observer)` is where we create our *handler factory* (the unique-per-request *observable subscription function factory*). Remember - this factory gives us back a unique *observable subscription function* for this request --xxx-- which is what we'll stash away in our map.

The second line calls `publish()`, which is an abstract method on the superclass that we must implement.  This is where we'll implement the custom details of our strategy.  Let's tackle that next.  Note that `send()` calls it with two parameters:
* the arguments from the user-land `client.send()` call (e.g., if the user wrote `client.send('/get-customers', {}), the first argument would contain `{pattern: '/get-customers', data: {}}`).
* the newly minted *observable subscription function factory*

#### The `ClientFaye#publish()` Method

This is where it all comes together.  Let's look at the whole method, then we'll break it down.

```typescript
// nestjs-faye-transporter/src/requestor/clients/faye-client.ts
protected publish(
    partialPacket: ReadPacket,
    callback: (packet: WritePacket) => any,
  ): Function {
    try {
      const packet = this.assignPacketId(partialPacket);
      const pattern = this.normalizePattern(partialPacket.pattern);
      const serializedPacket = this.serializer.serialize(packet);
      const responseChannel = this.getResPatternName(pattern);

      let subscriptionsCount =
        this.subscriptionsCount.get(responseChannel) || 0;

      const publishRequest = () => {
        subscriptionsCount = this.subscriptionsCount.get(responseChannel) || 0;
        this.subscriptionsCount.set(responseChannel, subscriptionsCount + 1);
        this.routingMap.set(packet.id, callback);
        this.fayeClient.publish(
          this.getAckPatternName(pattern),
          serializedPacket,
        );
      };

      const subscriptionHandler = this.createSubscriptionHandler(packet);

      if (subscriptionsCount <= 0) {
        const subscription = this.fayeClient.subscribe(
          responseChannel,
          subscriptionHandler,
        );
        subscription.then(() => publishRequest());
      } else {
        publishRequest();
      }

      return () => {
        this.unsubscribeFromChannel(responseChannel);
        this.routingMap.delete(packet.id);
      };
    } catch (err) {
      callback({ err });
    }
  }
```

Finally, let's talk about the call to `createSubscriptionHandler()`.  Back inside `publish`, we call this method as a factory to return the appropriate callback handler. So we're returning the handler that will be plugged into the `fayeClient.subscribe()` call &#8212; that is, the code that will be invoked for every inbound method.  This code wraps the call to our actual user-land handler.  It does the following:

1. Deserialize the packet.
2. Discard any mismatching packets\*
3. Destructure the message so we can deal with its constituents `err`, `response`, `isDisposed` and `id`
4. Use the map to lookup the correct callback handler
5. Discard any messages which are **not** destined for this client\**
6. Finally, return the results of the **actual** user-land handler call; we automatically add `isDisposed: true` if there's an error, to force the closure of the observable.

\* Blah
\** Blah

#### Connection Management

### What's Next

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
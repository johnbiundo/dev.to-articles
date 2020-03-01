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

For this article, you should `git checkout` the branch `part5`.  You can get more information [about the git repository branches here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-5-final-client-component).

#### Build the Apps for This Part

For each article in the series, we introduce some new components (and sometimes entirely new **projects**).  For convenience, at the start of each article, you should run the command\*

```bash
$ # from root directory of project (e.g., transporter-tutorial, or whatever you chose as the root)
$ sh build.sh
```
\**This is a shell script that runs on Linux-like systems.  You may have to make an adjustment for non-Linux systems.  Note that the script is simply a convenience that runs `npm install` inside each top-level directory, so you can always fall back to that technique if you have trouble with the script.*

### Overview

I ended the last article with a discussion of the shortcomings of our *Take 1* (first iteration) implementation of the Faye Custom Transporter client component.  In this article, we'll address those shortcomings.

### Understanding the Multiplexing Challenge

The *big problem* we found with our first implementation is that it's a little naïve about handling multiple simultaneous requests.  Let's make sure we understand why.  The first thing to think through is the premise of multiplexing.  What we're essentially doing is sharing a single pair of channels for sending and receiving unrelated requests.  Let's focus on the behavior of the response channel, as it seems like the potentially problematic one (after all, the request channel is just sort of "fire and forget").

We uncovered the problem in our "race test" at the end of the last chapter. The triggering scenario for our problem, in basic terms, is: **we sent two requests, where the second request finished before the first one**.  Our shared response channel just isn't prepared to handle this. It's well-behaved in the context of one-at-a-time requests: it subscribes and waits for a response, then unsubscribes. But in the face of multiple overlapping requests, its behavior is clearly incorrect. Let's try to describe why, as this will guide us to a solution.  The problem is this: we can have only a single active *Faye subscription handler*<sup>1</sup> no matter how many messages we send (this is by definition, since we have a single response channel).  Each request causes us to overwrite the previous *Faye subscription handler* when we re-issue the `fayeClient.subscribe(...)` call.  Moreover, that active one, with its built-in Observer, notifies **every** subscriber.

> <sup>1</sup>The generic term "subscription handler" is unfortunately overloaded in our discussion.  In the last chapter we introduced our nifty *observable subscriber function*.  Here we're talking about a *Faye subscription handler*.  These are two separate concepts, and we need to try to be very careful with our language.  **Especially** since we have been, and will continue to be, building objects that create a tight relationship between these two!  I'll do my best to keep the terminology straight, and you please do too!

### Strategy for Solving the Multiplexing Challenge

OK, let's start working on the strategy. Given the complexity of this problem, it's helpful to approach a solution starting from a somewhat abstract level, and successively refining our understanding and implementation.  We'll walk through the details of handling this in our Faye client component, but some of the details will vary from one transport library (e.g., broker) to the next because **each has its own message pattern and API**.  If you understand the concepts, there is a certain amount of boilerplate material that you can adapt to a particular transport library. Inevitably, if you need to build a new transporter client, you'll have to work through a few knotholes. In the mean time, pay more attention to the concepts than to understanding every line of code.

#### The Observable Part

We know `ClientProxy#send()` returns an Observable.  The job of that Observable is to consume incoming Faye messages (boxes in the diagram below) and emit them as a stream (circles in the diagram below) so that our user-land subscriber can deal with them as a nice, well-behaved RxJS Observable.

![Convert Messages to Stream](./assets/client-proxy-observable.gif 'Convert Messages to Stream')
<figcaption><a name="remote-stream"></a>Figure 1: Convert Messages to Stream</figcaption>

One valuable lesson we learned in the last chapter is that we can use the Observable/observer pattern to essentially "wrap our Faye client inside an Observable". This was our *observable subscriber function* pattern &#8212; implemented in the `handleRequest()` method ([review the source code here](xxx)) which builds the *observable subscriber function* (the bit of code that actually emits values through our Observable).  We're going to build on this lesson, and take it to the next level, to deal with the complexities of the multiplexing challenge.

#### Handling Multiple Requests: The Union of Observables and Correlation Ids

The problem we have to solve is how to associate a unique *observable subscription handler* with each Observable (i.e., with each `send()` request that returns that Observable as a response). Furthermore, we need to do this while having only a **single active Faye subscription handler**.  This is going to require a little higher order programming.  Stick with me &#8212; this is the hardest part of the tutorial, but we can get through it.

Let's propose defining our problem as follows: we are binding the *observable subscriber function* logic to our *Faye subscription handler* too soon/too statically.  Our solution needs to do late/dynamic binding of the *observable subscriber function* logic.  To be precise, it needs to delay binding the the *observable subscriber function* into the *Faye subscription handler* until a request is made so it can associate a unique one to each request. We have to make a little leap here.  Conceptually, what we're going to do is the following.  We'll make our single *observable subscriber function* logic dynamic by extracting a chunk of code into a factory function we'll call for each `send()` request. We'll store a unique instance of the function returned by this factory each time we create an Observable. Later, when we get a response, we'll retrieve that stored function and plug it back into the single static *observable subscribe function*, giving us a unique *observable subscriber function* for each request.  The factory function is actually provided by the framework:

```typescript
// from https://github.com/nestjs/nest/blob/master/packages/microservices/client/client-proxy.ts#L82
  protected createObserver<T>(
    observer: Observer<T>,
  ): (packet: WritePacket) => void {
    return ({ err, response, isDisposed }: WritePacket) => {
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
  }
```

Though the method name is `createObserver()`, I'm going to refer to it going forward as the *response emitter factory* to avoid confusion with the term *Observer*, which has a somewhat different meaning in RxJS. The function it returns, which we'll call the *response emitter*, is what we store and retrieve to make our *observable subscriber function* dynamic.

> Let's take a quick timeout to avoid terminology soup confusion!
>
> *Faye subscription handler*: A callback registered with the Faye client library that is invoked whenever a message with a particular topic is received.
>
> *Observable subscriber function*: A callback used during [Observable creation](xxx) to convert application events (e.g., inbound Faye messages) to Observer callbacks. For example, this is the mechanism that enables us to call the user-land handler with the value emitted by our Observer every time an inbound message is received.
>
> *Response emitter factor factory*: A [higher order function](xxx) that **returns** a *response emitter* (defined below)
>
> *response emitter*: A function that delivers the dynamic part of our *Faye subscription handler*.  We will have a unique instance of the *response emitter* for each Observable, overcoming the multiplexing problem.
>

The approach we'll take to implement this goes something like this:

1. At the time a user-land `send()` call is made, we'll use the *response emitter factory* to create a unique instance of the *response emitter*, along with a unique identifier\* (`id` field) for the request. We'll store an association between the unique id and the *response emitter* in a map.

    > \*At last, we see the genesis of the `id` field we've been carrying along with our messages all this time!
2. When a response is received, we'll extract the `id` from the response, and use it to look up the *response emitter*. We'll then call this unique instance of the *response emitter* from within the single shared *observable subscriber function*, and use this to emit the result.  Since the only Observable associated with this *response emitter* is the originator of the request, that's the only one that receives the emitted result.
3. To tie these things together, we need to pass the `id` all the way through the request/response flow.  The requesting code generates the `id` property, it's attached to the outbound message, and finally returned on the corresponding inbound message. Then we use it as described in step 2 above.

> The pattern we described in step 3 above is often referred to as using [correlation ids](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html) (for example, here's a [tutorial showing how](https://www.rabbitmq.com/tutorials/tutorial-six-python.html) a similar concept can be used for writing native RabbitMQ client apps).

#### Other Bookkeeping

One related issue we kind of glossed over is managing the Faye (response channel) subscription handler intelligently. We discussed the problem that we are essentially overwriting the *Faye subscription handler* each time we handle a new request. So an additional consequence of our design goal of **sharing a single inbound response channel** is that we we **must have only a single active subscription** to that response channel.  When all active requests have completed, we unsubscribe.  When a new request comes in, we subscribe again, and leave the subscription open until the channel quiesces (i.e., there are no more inflight requests) again.  We'll need to add some bookkeeping to improve our current behavior (of simply overwriting the handler with each new response).

#### Connection Management

One thing we'll find, as we integrate our code with the framework, is that we need to adhere to its expectations for how we provide a client library connection (i.e., the connection to the broker).  We'll mostly rely on some boilerplate code here, but let's briefly describe the strategy. Since we're packaging up a bunch of stuff inside *observable subscriber functions* (and their factories, etc.), the framework expects us to provide access to the connection in a particular way.  For the most part, we can just utilize some boilerplate to handle this.  There's only a small bit that is specific to a client library.  This is all packaged up in a `connect()` method that we must implement in our `ClientFaye` class.

#### Odds and Ends

We'll also see sprinkled in some functions that deal with other loose ends:
* we return channel names using a function, rather than hard coding them with the `_ack` and `_res` modifiers to avoid scattering magic strings throughout the code
* since user-land *patterns* can be arbitrary objects (we use strings in these articles, but Nest allows constructs like `client.send({cmd: 'get-customers'}, {})`), we use a built-in inherited function to "normalize" (essentially flatten) the pattern for internal usage

### Implementation

With this strategy in mind, here's the outline of how we'll implement it.

> Note that there's a fair amount of interaction back and forth between our `ClientFaye` subclass and the `ClientProxy` superclass it extends, so pay close attention to that, and I'll try to give clear signposts to guide you.

1. Unlike our *Take 1* version, where we created a new standalone class, we're now going to start by extending the `ClientProxy` class.  This is, of course, how we "plug in" to the framework.
2. We're going to let the framework take over the implementation of `send()` for us, rather than implement it ourselves as we did in Part 4. The superclass `send()` method is now the entry point for processing a user-land `send()` call, and dictates how we plug in our pieces without disrupting the overall machinery.
3. The superclass `send()` method calls upon a concrete implementation of `publish()`, which is where we'll implement the strategy we've been discussing, dealing with our *response emitter factory* construct, unique identifiers, and so forth, [as described above](#handling-multiple-requests-the-union-of-observables-and-correlation-ids).
4. We're going to implement a method to *unsubscribe* from a Faye topic.
5. We're going to provide a concrete implementation of `dispatchEvent()`, which is how `ClientProxy#emit()` is handled.
6. We're going to beef up *connection management*, as we mentioned above, to enable the framework to efficiently share our connection across multiple calls.
7. We'll take care of a few other minor details.

We have our shopping list, so let's get started!

### *Take 2* Code Review

#### The Superclass `send()` Method

Let's begin at a logical place, the `send()` method.  As mentioned, we'll now let the superclass handle this for us, and plug in our code appropriately.  Let's take a look:

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

Stripping this method to the bone, here's what it's doing:
1. Reusing a `connection` if it exists or obtaining a new one.  Note that since it depends on client library particulars to deal with obtaining a connection, it depends on the `connect()` method &#8212; which we are responsible for implementing and will get to soon.
2. Creating and returning the Observable that is effectively the "container" within which we'll implement the strategy we [discussed above](#strategy-for-solving-the-multiplexing-challenge).

Let's look at that Observable creation step for a moment.  The first line `const callback = this.createObserver(observer)` is where we create our *response emitter*.

The second line calls `publish()`, which is an abstract method on the superclass that we must implement.  This is where we'll implement the custom details of our strategy.  Let's tackle that next. But first, let's note that `send()` calls `publish()` with two parameters:
* the arguments from the user-land `client.send()` call (e.g., if the user wrote `client.send('/get-customers', {})`, the first argument would contain `{pattern: '/get-customers', data: {}}`).
* the newly minted *response emitter*

#### The `ClientFaye#publish()` Method

This is where it all comes together.  Let's look at the whole method, then we'll break it down.  For now, I suggest you read the comments and lightly scan the code below.  We'll dive into the code in a moment.

```typescript
// nestjs-faye-transporter/src/requestor/clients/faye-client.ts
protected publish(
    partialPacket: ReadPacket,
    callback: (packet: WritePacket) => any,
  ): Function {
    try {
      // prepare the outbound packet, and do other setup steps
      const packet = this.assignPacketId(partialPacket);
      const pattern = this.normalizePattern(partialPacket.pattern);
      const serializedPacket = this.serializer.serialize(packet);
      const responseChannel = this.getResPatternName(pattern);

      // start the Faye subscription handler "bookkeeping"
      let subscriptionsCount =
        this.subscriptionsCount.get(responseChannel) || 0;

      // build the call to publish the request; in addition to making the Faye API
      // `publish()` call, when this function is called, it:
      // * updates the bookkeeping associated with the Faye subscription handler
      //   count
      // * stashes our *response emitter* (called `callback` below) in the map
      const publishRequest = () => {
        subscriptionsCount = this.subscriptionsCount.get(responseChannel) || 0;
        this.subscriptionsCount.set(responseChannel, subscriptionsCount + 1);
        this.routingMap.set(packet.id, callback);
        this.fayeClient.publish(
          this.getAckPatternName(pattern),
          serializedPacket,
        );
      };

      // build the Faye subscription handler
      // this function retrieves the *response emitter*
      // and binds it to the rest of the Faye subscription handler logic
      const subscriptionHandler = this.createSubscriptionHandler(packet);

      // perform Faye `subscribe()` first (if needed), then Faye `publish()`
      if (subscriptionsCount <= 0) {
        const subscription = this.fayeClient.subscribe(
          responseChannel,
          subscriptionHandler,
        );
        subscription.then(() => publishRequest());
      } else {
        publishRequest();
      }

      // remember, this is an *observable subscriber function*, so it returns
      // the unsubscribe function.
      return () => {
        this.unsubscribeFromChannel(responseChannel);
        this.routingMap.delete(packet.id);
      };
    } catch (err) {
      callback({ err });
    }
  }
```

With the time we put in on the strategy discussion, this code should hopefully make sense, at least at a high level.  Let's first dispense with a couple of the minor details so they don't distract, then we can focus on the core functionality.
* At the top of the method (remember, this method is called synchronously when a user-land request is made), we prepare the outbound packet (the *request* message to be published), including assigning the packet `id` and serializing the packet.
* Subscription management should be straightforward to follow. We essentially keep a counter of *response channel* subscriptions. We (logically speaking) do a `count++` when publishing a request, and a `count--` when unsubscribing from the response channel. This let's us decide whether or not we need to subscribe to the response channel before publishing a request (solving our "we can only have one active subscription at a time" issue).

#### The `createSubscriptionHandler` Method: Binding the Response Emitter

Finally, let's talk about the call to `createSubscriptionHandler()` &#8212; first at a high level, and then the details. First, let's recognize that this a factory that returns the *actual Faye subscription handler* that gets bound to the Faye `subscribe()` call on the response channel (i.e., `<message-pattern>_res`).  In it, we **do the late binding** of the *observable subscription handler*. We do the late binding by looking up the *response emitter* by `id`, matching it with the request Observable, and calling it with the destructured inbound message fields.

With that in mind, we can explore a little further. The returned function has only one argument (as dictated by the [Faye API](https://faye.jcoglan.com/browser/subscribing.html)) &#8212; an actual Faye inbound message.  Here are the steps the *Faye subscription handler* that is produced from all this machinery performs:

1. Deserialize the packet.
2. Destructure the message so we can deal with its constituent values: `err`, `response`, `isDisposed` and `id`
3. Use the map to lookup the correct *handler factory*
4. Discard any messages which are **not** destined for this client<sup>1</sup>
5. Finally, return the **actual** *observable subscriber function*.  Note: we automatically add `isDisposed: true` if there's an error, to force the closure of the Observable.

> <sup>1</sup>This is the code fragment:
>  <code>if (!callback) {</code>
>  <code>&nbsp;&nbsp;return undefined;</code>
>  <code>}</code>
>
> We take this up in its own section immediately below.

#### Discarding Messages for Other Clients

Consider that in a distributed microservice-based architecture, we may have *multiple* clients (e.g., instances of `nestHttpApp` or other client apps), connected via the broker, to the same *responder* (e.g., `nestMicroservice`). In such a configuration, each client may issue the same requests (i.e., utilize the same message pattern).  When that happens, we'll have multiple clients communicating using the same Faye topic.  This is, of course, completely natural for Faye (and any message broker). However, since we have multiple subscribers on the same channel/Faye topic (e.g., `'/get-customers'`), **each** client (each subscriber) will be notified with any response message.  Only those that come from the requesting client will have a matching *handler factory*.  Others are properly destined to be handled by other client instances. We can simply ignore these, returning `undefined` to our *Faye subscription handler*, which is essentially a *no-op*.

#### Connection Management

### Event Handling

The final feature to implement is *event handling* &#8212; handling user-land `client.emit(...)` calls.  As we found on the server side in Part 3, this is far simpler than dealing with *request/response* because we are dealing with one-way messages: we simply send a message over the outbound (request) channel, and we're done.  No subscribing to responses necessary.

Because this is straightforward and follows a predictable pattern, the framework handles most of this for us in the `ClientProxy#emit` method.  Here's what that code looks like:

```typescript
  public emit<TResult = any, TInput = any>(
    pattern: any,
    data: TInput,
  ): Observable<TResult> {
    if (isNil(pattern) || isNil(data)) {
      return _throw(new InvalidMessageException());
    }
    const source = defer(async () => this.connect()).pipe(
      mergeMap(() => this.dispatchEvent({ pattern, data })),
      publish(),
    );
    (source as ConnectableObservable<TResult>).connect();
    return source;
  }
```

Here, once again, the framework is handling connection management for us, and all we really need to do is provide a concrete implementation for `dispatchEvent()`.  The superclass defines an abstract `dispatchEvent` method as follows:

```typescript
protected abstract dispatchEvent<T = any>(packet: ReadPacket): Promise<T>;
```

So our implementation simply needs to accept a request packet and publish the appropriate message.  We need to wrap that in a promise to match the signature above.

Here's how we implement the `dispatchEvent()` method in our client:

```typescript
  protected dispatchEvent(packet: ReadPacket): Promise<any> {
    const pattern = this.normalizePattern(packet.pattern);
    const serializedPacket = this.serializer.serialize(packet);

    return new Promise((resolve, reject) =>
      this.fayeClient.publish(pattern, serializedPacket),
    );
  }
```

The logic should be easy to follow.  We simply extract the pattern, normalize it, serialize the outbound packet (no need for an `id` on this one), and then return a Promise that resolves to the results of the `fayeClient.publish()` call.  The framework handles the rest (which is really very little other than efficient connection management).

### Loose Ends

* unsubscribe
*

### Acceptance Testing


### Conclusion

### What's Next

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
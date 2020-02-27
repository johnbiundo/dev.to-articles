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

### Leveraging the Framework

Well, we've come to it.  A point where we're going to put all our accumulated knowledge thus far to the test. I pulled a bit of a fast one on you in [Part 4](xxx). Not literally, but in terms of what you might be expecting in this article.  The step from [Part 2 (first iteration of the server)]() to [Part 3 (second iteration)]() was comparatively small.  The step from [Part 4]() to this one is quite a bit bigger.  It turns out that managing a multiplexing client is a non-trivial challenge. The core requirements haven't really changed, so what we learned in [Part 4]() still very much applies, but we're going to have to roll up our sleeves in this chapter and dial up some serious code.  Wait! Don't panic!

There's good news! The framework handles most of this for us.  What we are really going to have to do is figure out how to plug our broker-specific stuff **into** the framework so we can leverage its capabilities and keep our code fairly simple.  Let's get to it.

### Addressing the Multiplexing Challenge

When it comes right down to it, the conceptual solution to the multiplexing challenge is straightforward.  We've already seen that we associate a unique identifier with each outbound message, and for responses, the server copies that identifier into the response message. This gives us end-to-end traceability of each unique request as it traverses the system. This is a common technique, sometimes known as using a [correlation id](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html) (for example, here's a [tutorial showing how](https://www.rabbitmq.com/tutorials/tutorial-six-python.html) a similar concept can be used for writing native RabbitMQ client apps).  The question is, how do we use these?  You probably already suspect the answer: keep a map of `id -> request handler instance`, and make sure when a response comes back, we direct it to the same handler instance that initiated the request.

As I mentioned, there's good news here: the framework has a bunch of the machinery to handle this. In addition to helping us manage the multiple outstanding requests, it takes care of connection management and subscription management --xxx-- I'll talk about these in more detail in a little while.  To avail ourselves of this built-in infrastructure, we have to get into the "framework" mindset: that is, invert our thinking and figure out where we need to supply the correct code to be **called by the framework**.  Turns out, the list of things we need to do isn't too long, nor too hard to [grok](https://en.wikipedia.org/wiki/Grok):

1. We're going to extend the `ClientProxy` class instead of creating a new standalone class.
2. We're going to let the framework take over the implementation of `send()` for us.  That's the entry point for processing a user-land `send()` call, and allows us to plug in our pieces without disrupting the overall machinery.
3. The superclass `send()` method expects a concrete implementation of `publish()`, which represents our [now-familiar](xxx) *subscribe-to-the-response-then-publish-the-request* pattern.
4. We're going to create a method to *unsubscribe* from a Faye topic.
5. We're going to provide a concrete implementation of `dispatchEvent()`, which is how `ClientProxy#emit()` is handled.
6. We're going to beef up *connection management* (how we stay connected to the Faye broker) to enable the framework to efficiently share our connection across multiple calls
7. We'll take care of a few other minor details.

We have our shopping list, so let's get started!  There's a fair amount of interaction back and forth between the superclass and the `ClientFaye` class we're writing to extend it, so pay close attention to that, and I'll try to give clear signposts to guide you.

### *Take 2* Code Review

#### The Superclass `send()` Method

Let's start with the superclass `send()` method. Since we're going to be referring to this a lot, I'm going to invent an acronym to describe our *subscribe-to-the-response-then-publish-the-request* technique: we'll call it *STRTPR*.  As a very fast reminder [refresh the details here](), this is the pattern whereby, to handle a request/response message, the client:
1. subscribes to the response channel ("subscribe-to-response" = *STR* part of our acronym)
2. publishes a request on the request channel ("publish-the-request" = *PTR* part of our acronym)

Here's the code:
```typescript
// the ClientProxy superclass.  See source code here:
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

Stripping the superclass `send()` method to the bone, what it's doing is:
1. Sharing an existing `connection` (we are responsible for implementing `connect() --xxx-- we'll get to that), or obtaining a new one.
2. Creating the Observable that contains our *STRPTR* (which we'll build inside a concrete implementation of `publish()`). If you look inside [`ClientProxy#createObserver`](https://github.com/nestjs/nest/blob/master/packages/microservices/client/client-proxy.ts#L82), you'll see it's **very similar** to the [observer creation code](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/part4/nestjs-faye-transporter/src/requestor/clients/faye-client.ts#L34) we built in the last chapter, which should make you feel comfortable with what it's doing.  In a nutshell, we're abstracting that same STRPTR pattern we used up one level (one order higher in our higher order programming stack) so that we can introduce a mechanism to associate individual handler instances with their correlation ids

#### The `ClientFaye#publish()` Method

Get comfortable.  Maybe get a cup of coffee or tea. We're going to be here a while :smiley:.

Take a moment to look over the whole method:

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

This is where everything pretty much coalesces to implement our multiplexed requests.  Again, this bears quite a bit of similarity to the `handleRequest()` method ([code here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/part4/nestjs-faye-transporter/src/requestor/clients/faye-client.ts#L27)) we built in the previous article.  That's not surprising, as it's implementing our *STRPTR*.  If you look at it from 10,000 feet, you can see the outlines of that: we build a `publish()` call, we then subscribe to the response channel (with an appropriate handler), we then publish the request.  As we did in the last chapter, we return an `unsubscribe()` function as part of the Observable creation.

Nestled in amongst that are two bits of what we might call "accounting":
1. Keeping track of our *open subscriptions* on the response channel.  We do this because there is only **one** physical response channel.  When all outstanding requests are satisfied, we will unsubscribe.  If there are outstanding requests, we know a subscription on the **one** (shared) response channel is already in place.
2. Managing that `id -> response handler instance` map we talked about earlier.  This is managed in `this.routingMap`, a JavaScript [map]() we inherit from the superclass.  Each new request adds an entry to the map.  Each completed response deletes the entry.

### What's Next

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
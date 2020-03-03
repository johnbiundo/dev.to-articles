---
published: False
title: "Part 4: Basic Client Component"
description: "tags: nestjs, nest, faye, microservices, node.js"
series: "Advanced NestJS Microservices"
cover_image: "https://dev-to-uploads.s3.amazonaws.com/i/lgt7m4a8dton9rvuimje.gif"
canonical_url:
---

*John is a member of the NestJS core team*

### Introduction

This is Part 4 of a six-part series.  If you landed here from Google, you may want to start with [Part 1](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l).

In this article, we build the first iteration of the client component of the Faye Custom Transporter.

**Reminder**: Many of the concepts and terminology here are introduced and explained in [this article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3). That series serves as a good foundation for understanding the more advanced concepts covered in this series.

#### Get the Code

All of the code in these articles is available [here](https://github.com/johnbiundo/nestjs-faye-transporter-sample).  As always, these tutorials work best if you follow along with the code.  The [README](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md) covers all the steps you need to get the repository, build the apps, and follow along.  It's easy!  I strongly encourage you to do so.  Note that each article has a corresponding branch in the repository.  For example, this article (Part 4), has a corresponding branch called `part4`. Read on, or [get more details here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-4-initial-client-component) on using the repository.

#### Git checkout the current version

For this article, you should `git checkout` the branch `part4`.  You can get more information [about the git repository branches here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-4-initial-client-component).

#### Build the Apps for This Part

For each article in the series, we introduce some new components (and sometimes entirely new **projects**).  For convenience, at the start of each article, you should run the following command from the top-level directory (where you cloned the repo): \*

```bash
$ # from root directory of project (e.g., transporter-tutorial, or whatever you chose as the root)
$ sh build.sh
```
\**This is a shell script that runs on Linux-like systems.  You may have to make an adjustment for non-Linux systems.  Note that the script is simply a convenience that runs `npm install && npm run build`inside each top-level directory, so you can always fall back to that technique if you have trouble with the script.*

### Overview

It's time to turn our attention to the client component. But let's start by recalling that we spent a lot of effort ensuring that the server component can return an observable stream of results to a requestor.  Let's have a quick visual to cement that idea, since we'll start right out with consuming that stream on the client side.

![Remote Observable Stream](./assets/remote-stream3.gif 'Remote Observable Stream')
<figcaption><a name="remote-stream"></a>Figure 1: Remote Observable Stream</figcaption>
### What's Next

In this animated sequence, our `nestMicroservice` *responder* app responds to a *request* message (`ClientProxy#send`) by emitting a stream of three results (circles).  We saw in the last article how our transporter server component converts that stream to a sequence of 3 messages (boxes) to the broker, sending those messages on the response channel.  Our first meaningful task with the client component will be to consume those broker messages and convert them back into an Observable stream (circles) in the `ClientProxy` layer so that our user-land controllers and services can use them as regular **RxJS** observables.

In this iteration (*Take 1*), we'll take a similar approach to our first iteration of the server.  We'll get a basic functioning client working so we can focus on its responsibilities and the main code path.  Later, we'll add more robustness.  As with the server side, we'll ignore events (the `ClientProxy#emit` path) for now.

### First Iteration (Take 1) of the Client Component

Let's get started.  First, let's look at how someone would use the client component of our Faye Custom Transporter.  Open the `app.controller.ts` file for our `nestHttpApp`.

```typescript
// nestHttpApp/src/app.controller.ts
import {
  Controller,
  Get,
  Logger,
  Param,
  Response,
  Post,
  Body,
} from '@nestjs/common';

import {
  ClientFaye,
  InboundResponseIdentityDeserializer,
  OutboundMessageIdentitySerializer,
} from '@faye-tut/nestjs-faye-transporter';

@Controller()
export class AppController {
  logger = new Logger('AppController');
  client: ClientFaye;

  constructor() {
    this.client = new ClientFaye({
      url: 'http://localhost:8000/faye',
      serializer: new OutboundMessageIdentitySerializer(),
      deserializer: new InboundResponseIdentityDeserializer(),
    });
  }
```

We simply instantiate the client in the class constructor.  We could also use [constructor injection](https://docs.nestjs.com/microservices/basics#client).  **Note**: one thing we can **not** currently do is use the `@Client()` decorator.  That's not really a significant limitation, as the decorator is merely a convenience, and is not the preferred way of instantiating a `ClientProxy`.

The JSON options object we pass to `ClientFaye` should be quite similar to the `options` object we support on the server side, as it is either configuring the Faye connection, or passing some generic transporter options, like `serializer`.

Once we instantiate a client like this, we can use it just like a `ClientProxy` object to make requests and emit events. Let's take a look at one such request.  Further down in the `app.controller.ts` file, notice this call.  Here, we make a call to `client.send()` and &#8212; since the result will emit a stream &#8212; we pipe that stream through some RxJS operators.

```typescript
// nestHttpApp/src/app.controller.ts
@Get('jobs-stream1/:duration')
  stream(@Param('duration') duration) {
    return this.client.send('/jobs-stream1', duration).pipe(
      // do notification
      tap(step => {
        this.notify(step);
      }),
    );
  }
```

### Take 1 Requirements

The request above is the one represented in the animation at the top of the article.  Feel free to take a look at the server-side code (`nestMicroservice/src/app.controller.ts`) to get familiar with what's happening here.  You can run this now by issuing an HTTP request to the `nestHttpClient` app as shown below.  Assuming you've got your multi-terminal setup going, you'll be able to follow the full flow of this request and response through all of the components.

```bash
$ # HTTPie command to request `/jobs-stream1` with base duration of 1
$ http get localhost:3000/jobs-stream1/1
```

You can also read the [deep dive on Observables](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/observable-deepdive.md) for a thorough treatment of the elegance of streaming server side observable results back to the client.  For now, all we need to know is what we see in the animation above: when we make this request, we get a *stream of results* in the form of a sequence of messages.

Let's define our acceptance criteria for *Take 1* as follows.  Our `ClientFaye` object should let us send the `'/jobs-stream1'` request, and handle the response, converting the message stream back into an Observable.

> In [part5](https://dev.to/nestjs/part-5-completing-the-client-component-hlh-temp-slug-2907984?preview=82c11163db963ca01d8d62d3a7b14843b422a6b28f46762d999bbe4b7035ad634d48bbbdd740e36376121aa673354ff5259f8b3028bceb931e800d9e) we'll complete the implementation and have a fully functioning Faye Custom Transporter &#8212; both client and server side.

### Take 1 Strategy

Before diving into the code, let's review our overall strategy.  At a top level, it's pretty simple. For each user-land *request* (e.g., the `this.client.send(...)` call in the `app.controller.ts` file shown above), we want to issue the remote request and wait for the results, converting the response message(s) into an Observable stream.

Issuing the request and waiting for the results using the broker API is familiar territory.  We'll just implement our very familiar *STRPTQ* (*subscribe-to-the-response-then-publish-the-request*) pattern to accomplish this.

How do we convert the broker message stream into an Observable stream? We can use the Observable/observer pattern to do this. While I won't go into a lot of detail (you can [read more, including simple examples, here](xxx)), the basic idea is simple, and is the primary use case for Observables.  The easiest way to see this is through the code review below.

### Take 1 Code Review

Let's dive into the code.  Open the file `nestjs-faye-transporter/src/requestor/clients/faye-client.ts`.

You might notice that in this iteration, we're not actually extending any framework class.  We'll need to do that in the next iteration, but it makes our code smaller and easier to focus on the core logic if we omit that in this step and just define a new class.

#### Class Definition

Let's start with the class members and constructor:

```typescript
// nestjs-faye-transporter/src/requestor/clients/faye-client.s
export class ClientFaye {
  private readonly serializer;
  private readonly deserializer;
  protected fayeClient;

  constructor(protected readonly options) {
    this.fayeClient = this.connect();
    this.serializer = options.serializer || new IdentitySerializer();
    this.deserializer =
      options.deserializer || new IncomingResponseDeserializer();
  }
```

We should be familiar with the `serializer` and `deserializer` properties, and how they're initialized, from the server side.  The `fayeClient` holds our connection to the Faye broker and is the object we'll use to make Faye client API calls.  The constructor just sets this stuff up.  It defaults to using the built-in (essentially **no-op**) serializer and deserializer if the user doesn't provide one in the options.

#### Implementing `send()`

With that covered, we can look at the API our class exposes.  First, recall that the user-land `send()` call we need to support is:

```typescript
// nestHttpApp/src/app.controller.ts
@Get('jobs-stream1/:duration')
  stream(@Param('duration') duration) {
    return this.client.send('/jobs-stream1', duration).pipe(
      // do notification
      tap(step => {
        this.notify(step);
      }),
    );
  }
```

We expose this feature, of course, with the `send()` method right at the top of the `faye-client.ts` file, as shown here:

```typescript
// nestjs-faye-transporter/src/requestor/clients/faye-client.ts
  public send(pattern, data): Observable<any> {
    return new Observable(observer => {
      return this.handleRequest({ pattern, data }, observer);
    });
  }
```

#### The `send()` Method Returns an Observable

As required, `send()` returns an `Observable`. We need to delve into Observable-land here for a while. This can be confusing/frustrating if you're not familiar with the topic, but by the time you're finished here, I *think* the :bulb: will go on.

In the `send()` method body, we construct an Observable using the *normal Observable creation pattern*<sup>1</sup> and return it to user-land.  Applying the normal Observable creation pattern in our context means we pass the constructor a function that **accesses the Faye message stream and emits the values through our Observable**. Traditionally, this callback is called the *observable subscriber function*.

Simple Observables (the ones you'll find in a lot of Medium articles :wink:) often define this *observable subscriber function* body right in place.  For example:

```typescript
return new Observable(observer => {
  observer.next('hello');
  observer.next('world');
  observer.complete();
});
```

But we already know that ours is going to be a little complex because the **source** of the data we are going to emit comes from the Faye broker.  Specifically, it comes from the callback we're going to register in the Faye client API `subscribe()` call which is invoked each time a message comes in on the response channel.  **That** code &#8212; which listens through Faye for inbound messages &#8212; is the code that needs to run our `observer.next()` (or equivalent) function call.

In these more complex scenarios, to keep the code clean, we delegate this sophisticated *observer subscriber function* to a specialized function. That's what we're doing in the code above by calling `handleRequest()`.

For the moment, you need to know two things:
1. `handleRequest()` **is** our *observable subscriber function*
2. `handleRequest()` is going to make a Faye client API call to subscribe to the response topic, causing it to register to receive future inbound requests.  As it receives them, it's going to emit the results in an Observable stream, analogous to our `observer.next('hello')` statement above.  We'll see that code in a minute, but let's keep it abstract for another minute.

This can all seem kind of strange if you're not familiar with the Observable pattern<sup>1</sup>. Let's see if it helps to think about it this way.  We're setting up callbacks at several levels in a sort of cascade.  At the "innermost" level, we have the *Faye subscription handler* callback that will run when we get an inbound message on the response channel.  When this handler runs, we want to in turn trigger a callback **through the Observer mechanism** to a higher-level callback &#8212; the user-land response handler (the code that looks like `pipe(tap(step) => {...})` in the `app.controller.ts` file, shown above).

Here are the connections you need to make:

* inner callback (broker) called when inbound message received...
* this triggers calling the *observable subscription function*...
* which in turn *calls back to* our user-land `client.send(...)` subscription

That's what the `send()` method is setting up with the Observable it creates.  One more thing about Observables to keep in mind.  What we just described effectively "sets up the cascade of callbacks", but these functions are **cold** in the sense that all we've done is register callbacks. The act of **subscribing** to the Observable kicks things in motion. The act of subscribing is the event that triggers the *observable subscriber function* to run; when you think about that, you realize that "Aha! So this in turn causes the Faye API `subscribe()` call inside `handleRequest()` to run", and we see that this is how we start the stream flowing and turn the process from **cold** to **hot**.

Another thing to know is that if the user-land `client.send()` call doesn't **explicitly** subscribe, but instead just **returns** the result, Nest **automatically** subscribes to the result. As a `ClientProxy` developer, you don't have to worry about this.  But as a quick aside, note that this behavior is useful as a default, and works great in most situations.  But in case you're wondering *how does Nest return a potential stream of results* over HTTP?  The answer is that it returns the **last** result in the stream.  Again, in many cases this is fine.  In our example above (shown again below), however, we want to access the values in the stream, hence we add a `pipe()` operator that allows us to operate on the stream itself, once it's subscribed:

```typescript
// nestHttpApp/src/app.controller.ts
@Get('jobs-stream1/:duration')
  stream(@Param('duration') duration) {
    return this.client.send('/jobs-stream1', duration).pipe(
      // do notification
      tap(step => {
        this.notify(step);
      }),
    );
  }
```

If this still isn't completely intuitive, I feel your pain, but I advise you to just let it marinate in the background and proceed.  Trust me, you can still follow most of this even if this still feels *strange*.  If you really **must scratch that itch**, see the footnote below.

> <sup>1</sup>I'm assuming familiarity with this pattern. If you're an RxJS junky, this should be easy to follow.  If you're not, I'll try not to wave my hands too much. I too struggled with this concept for a while, so I've created a small [side excursion here](xxx) to help with this concept.  This should take you no more than 5-10 minutes to work through, and might be a helpful pre-requisite if some of this section has you a little boggled.

So let's dig into that `requestHandler()` method.

#### The Request Handler

The `handleRequest()` method is where we marry up our Observable pattern with our *request/response* handling pattern. It's easiest if we decompose this and look at a version that abstracts out the Observable code first.  Then we'll see how it all wires together.

In the first pseudocode-ish version below, we can focus directly on the *request/response* handling pattern.  You'll quickly recognize this as our old friend the **STRPTQ** pattern. (As a refresher, feel free to review how we implemented this functionality in our native Faye `customerApp` way back in [Episode 1](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l).  Here's a [link to that code](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/part4/customerApp/src/customer-app.ts#L20).  Or just open up `customerApp/src/customer-app.ts`).

With that in mind, the pseudocode-ish version of `handleRequest()` below should be pretty clear.  I'll make a few explanatory comments below the code sample.

```typescript
  public handleRequest(partialPacket, observer): Function {
    const packet = Object.assign(partialPacket, { id: uuid() });
    const serializedPacket = this.serializer.serialize(packet);

    const requestChannel = `${packet.pattern}_ack`;
    const responseChannel = `${packet.pattern}_res`;

    const subscriptionHandler = ... /* we'll fill this in shortly!  This function
      will provide the code we want to plug in as our *Faye subscription handler*.
      Guess what?  This is where we'll wire in our Observable :)
    */

    // Execute the familiar subscribe-to-response-then-publish-the-request pattern
    const subscription = this.fayeClient.subscribe(
      responseChannel,
      subscriptionHandler,
    );

    subscription.then(() => {
      this.fayeClient.publish(requestChannel, serializedPacket);
    });

    /*
      Handle any other Observable requirements
    */
  }
```

A few other notes from the code:

* The `partialPacket` parameter has most of the elements of our outbound request message.  It's been invoked from `send()` with an object like:
    ```json
    {
      pattern: '/jobs-stream1/',
      data: 1
    }
    ```

    We complete the formation of our outbound packet by adding that pesky `id` parameter (that we **still** haven't talked about &#8212; I promise to address this in [Part 5](https://dev.to/nestjs/part-5-completing-the-client-component-hlh-temp-slug-2907984?preview=82c11163db963ca01d8d62d3a7b14843b422a6b28f46762d999bbe4b7035ad634d48bbbdd740e36376121aa673354ff5259f8b3028bceb931e800d9e)!).

* After running our serializer, the request packet is ready to send.
* In Faye, the `subscribe()` call returns a promise that resolves when the Faye server returns an acknowledgement.
* Upon that acknowledgement (promise resolution), we publish the serialized packet we built earlier.

Now let's deal with marrying that up with our Observable.  Here's the final version of `handleRequest()`.

```typescript
  public handleRequest(partialPacket, observer): Function {
    const packet = Object.assign(partialPacket, { id: uuid() });
    const serializedPacket = this.serializer.serialize(packet);

    const requestChannel = `${packet.pattern}_ack`;
    const responseChannel = `${packet.pattern}_res`;

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

    const subscription = this.fayeClient.subscribe(
      responseChannel,
      subscriptionHandler,
    );

    subscription.then(() => {
      this.fayeClient.publish(requestChannel, serializedPacket);
    });

    return () => {
      this.fayeClient.unsubscribe(responseChannel);
    };
  }
```

Let's focus in on the newly added part:

```typescript
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
```

Let's review the purpose of this chunk of code.  This is (in terms of the concepts we started out with), our **inner callback** &#8212; the function that will be passed in our Faye client API `subscribe()` call to handle inbound messages.  As we handle each message in this function, we are converting the sequence back into an Observable stream.  We do that by taking the following steps **for each inbound message received from Faye**:

1. Deserialize it and destructure it, leaving us with an `error`, `response` and `isDisposed` value for each inbound message
2. Based on the contents of the message, determine if the **inbound message stream** has errored or is complete
3. Emit an observable value, based on step 2

As in any normal *observable subscriber function*, we handle these cases with the `observer.error()`, `observer.next()` and `observer.complete()` calls. If this confuses you, remember that all of this plugs into the Observable we created in our `send()` method. You might take a minute to [grok](https://en.wikipedia.org/wiki/Grok) all of this, and if it's still a little fuzzy, you can visit my [mini Observable factory tutorial here](xxx).

#### Handling Unsubscribe

Any *observable subscriber function* must return a way to unsubscribe from the Observable.  This is part of the Observable creation pattern.  The unsubscribe method can be called by the user; more often, it is called automatically when the Observable stream ends (e.g., when the final message is returned from our remote responder).

The last bit of code in `handleRequest()` fulfills this contract for our *observable subscriber function*:

```typescript
    return () => {
      this.fayeClient.unsubscribe(responseChannel);
    };
```

The net effect of this is that when the stream completes (e.g., when the inbound message has `isDisposed` set to true), we unsubscribe from the response channel.

#### The Rest of the Code

The only other thing left is the connection management to the Faye broker.  That's handled in the `connect()` method, which for now we're calling in the constructor.  This should be pretty understandable.  The only slightly funky thing is that we use destructuring to extract out the `url` and the `options` object from the constructor options so we can easily format our call to `fayeClient.connect()`.

Connection management will become more important and consequently more sophisticated in our next iteration.

```typescript
  public connect() {
    if (this.fayeClient) {
      return this.fayeClient;
    }
    const { url, serializer, deserializer, ...options } = this.options;
    this.fayeClient = new faye.Client(url, options);
    this.fayeClient.connect();
    return this.fayeClient;
  }
```

### Acceptance Testing

We're ready to test our code.  In previous articles, I gave a lot of suggestions about managing your multiple terminals.  Going forward, I'll simply indicate the steps you should take, and assume you've got a working environment with the appropriate logical terminals set up.  Here are the steps to run our test:

* Make sure our Faye server is up and running in one terminal.
* Make sure the `nestMicroservice` app is running in one terminal.
* Make sure the `nestHttpApp` app is running in one terminal.

Now, issue a request (expressed below as an HTTPie request, but use your favorite HTTP testing tool):

```bash
$ # HTTPie request to get the jobs stream with a base duration of 1
$ http get localhost:3000/jobs-stream1/1
```

If all is working, you should get a coordinated set of logs in each of the terminal windows indicating that we have, indeed, successfully subscribed to the results of our remote request. You'll see our RxJS pipe come into play, making simulated calls to our notifier as each step in the job completes (as each message in the stream arrives). :beer:!

### Limitations of Take 1

We knowingly deferred a few things to *Take 2* to keep focused on the core requirements and implementation of a client. We'll address them in Part 5.  But there's one **big** issue lurking.  Any guesses on what that might be? I've definitely foreshadowed this along the way (**hint**: repeatedly deferring the discussion of that pesky message `id`).

OK, let's get right to it. Ask yourself this: since our client is embedded in an HTTP app that is itself responding to multiple incoming requests, many of which in turn are triggering these `client.send(...)` remote requests, how does the transporter client keep track of all of the outstanding remote requests and route them back to the correct originating (HTTP) request handler (i.e., the code with the original `client.send()` call?

Let's try it out. You'll need one extra terminal window for this to enable launching two simultaneous requests.  You'll notice that in this branch, there is an extra route in `nestHttpApp`.  That route makes a new request (`'/race'`), which has a corresponding `@MessagePattern()` in `nestMicroservice`.  The point of these new features is to enable us to launch multiple requests that **overlap** in their execution.  Feel free to take a look at the code (these new routes/handlers are at the bottom of their respective Controller files), but all you really need to know is this: the `race` route in our `nestHttpApp` takes two parameters:
* a requestId
* a delay period (in seconds) specifying how long the server should wait before responding

For example, try making the following request (HTTPie format):

```bash
$ http get localhost:3000/race/1/10
```

This will return the following response after 10 seconds (`cid` in the response corresponds to the first route parameter, and `delay` in the response corresponds to the second parameter (expressed in milliseconds)):
```json
{
    "cid": "1",
    "customers": [
        {
            "id": 1,
            "name": "fake"
        }
    ],
    "delay": 10000
}

```

With this in place, we can run two overlapping requests. Let's go ahead and run the following two requests.  You'll need to do this in two separate terminals, and you should launch the second request within 5 seconds of launching the first (to achieve the desired overlap):

```bash
$ # run this in terminal 1, and be prepared to run the second one in terminal 2 a few seconds later
$ http get localhost:3000/race/1/10
$ # run this one in terminal 2, within a couple of seconds of running the top 1
$ http get localhost:3000/race/2/5
```

The results may (well, maybe not with all the foreshadowing :wink:) surprise you.  **Both** requests return the following result:

```json
{
  "cid": "2",
  "customers": [
      {
          "id": 1,
          "name": "fake"
      }
  ],
  "delay": 5000
}
```

This is obviously incorrect for the first request.  The issue demonstrates what I'll refer to as the *[multiplexing](http://250bpm.com/blog:18) challenge*.  That is, we are multiplexing our requests for each *pattern* through a single pair of channels.  When we issue request #1, it subscribes to a response on the `'/race_res'` channel. Request #2 does the same thing, and since request #2 finishes first, the request #1 subscription gets **request #2's result** (the wrong answer!) on the response channel.

Also, there's another nagging question... why do **both** subscribers get the result?  Hmmm... clearly something isn't working quite right here. We'll have to dig and work on this in our next (and final!) revision!

### What's Next

With these issues in mind, we're ready to build a final version of the Faye Custom Transporter client component, completing our development project.  In Part 5 we cover:

* Solving the multiplexing issue
* Client library connection management
* Event handling on the client side

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
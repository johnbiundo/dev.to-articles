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

This is part 4 of a six-part series.  If you landed here from Google, you may want to start with [part 1](https://dev.to/nestjs/build-a-custom-transporter-for-nestjs-microservices-dc6-temp-slug-6007254?preview=d3e07087758ff2aac037f47fb67ad5b465f45272f9c5c9385037816b139cf1ed089616c711ed6452184f7fb913aed70028a73e14fef8e3e41ee7c3fc#requestresponse).

In this article, we build the first iteration of the client component of the Faye Custom Transporter.

**Reminder**: Many of the concepts and terminology here are introduced and explained in [this article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3). That series serves as a good foundation for understanding the more advanced concepts covered in this series.

#### Get the Code

All of the code in these articles is available [here](https://github.com/johnbiundo/nestjs-faye-transporter-sample).  As always, these tutorials work best if you follow along with the code.  The [README](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md) covers all the steps you need to get the repository, build the apps, and follow along.  It's easy!  I strongly encourage you to do so.  Note that each article has a corresponding branch in the repository.  For example, this article (part 4), has a corresponding branch called `part4`. Read on, or [get more details here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-4-initial-client-component) on using the repository.

#### Git checkout the current version

For this article, you should `git checkout` the branch `part4`.  You can get more information [about the git repository branches here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-4-initial-client-component).

#### Build the Apps for This Part

For each article in the series, we introduce some new components (and sometimes entirely new **projects**).  For convenience, at the start of each article, you should run the command\*

```bash
$ # from root directory of project (e.g., transporter-tutorial, or whatever you chose as the root)
$ sh build.sh
```
\**This is a shell script that runs on Linux-like systems.  You may have to make an adjustment for non-Linux systems.  Note that the script is simply a convenience that runs `npm install` inside each top-level directory, so you can always fall back to that technique if you have trouble with the script.*

### Overview

It's time to turn our attention to the client component. But let's start by recalling that we spent a lot of effort ensuring that the server component can return an observable stream of results to a requestor.  Let's have a quick visual to cement that idea, since we'll start right out with consuming that stream on the client side.

![Remote Observable Stream](./assets/remote-stream3.gif 'Remote Observable Stream')
<figcaption><a name="remote-stream"></a>Figure 1: Remote Observable Stream</figcaption>
### What's Next

In this animated sequence, our `nestMicroservice` *responder* app responds to a *request* message (`ClientProxy#send`) by emitting a stream of three results (circles).  We saw in the last article how Nest converts that stream to a sequence of 3 messages (boxes) to the broker on the response channel.  Our first meaningful task with the client component will be to consume those messages and convert them back into an observable (circles) in the `ClientProxy` layer so that our user-land controllers and services can use them as regular **RxJS** observables.

In this iteration (*Take 1*), we'll take a similar approach to our first iteration of the server.  We'll get a basic functioning client working so we can focus on its responsibilities and the main code path.  Later, we'll add more robustness.  As with the server side, we'll ignore events (`ClientProxy#emit`) for now.

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

The options object we pass to `ClientFaye` should be quite similar to the options object we support on the server side, as it is either configuring the Faye connection, or passing some generic transporter options, like `serializer`.

Once we instantiate a client like this, we can use it just like a `ClientProxy` object to make requests and emit events. Let's take a look at one such request.  Further down in the `app.controller.ts` file, notice:

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

The request above is the one we saw in the animation at the top of the article.  Feel free to take a look at the server-side code (`nestMicroservice/src/app.controller.ts`) to get familiar with what's happening here.  You can run this now by issuing an HTTP request to the `nestHttpClient` app as shown below.  Assuming you've got your multi-terminal setup going, you'll be able to follow the full flow of this request and response through all of the components.

```bash
$ # HTTPie command to request `/jobs-stream1` with base duration of 1
$ http get localhost:3000/jobs-stream1/1
```

You can also read the [deep dive on Observables](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/observable-deepdive.md) for a thorough treatment (and a *treat* as well :smiley:).  For now, all we need to know is what we see in the animation above: when we make this request, we get a *stream of results* in the form of a sequence of messages.

Let's define our acceptance criteria for *Take 1* as follows.  Our `ClientFaye` object should let us send the `'/jobs-stream1'` request, and handle the response, converting the message stream back into an observable.

> **Note**: in [part5](https://dev.to/nestjs/part-5-completing-the-client-component-hlh-temp-slug-2907984?preview=82c11163db963ca01d8d62d3a7b14843b422a6b28f46762d999bbe4b7035ad634d48bbbdd740e36376121aa673354ff5259f8b3028bceb931e800d9e) we'll complete the implementation and have a fully functioning Faye Custom Transporter &#8212; both client and server side.

### Take 1 Code Review

Let's dive into the code.  Open the file `nestjs-faye-transporter/src/requestor/clients/faye-client.ts`.

You might notice that in this iteration, we're not actually extending any framework class.  We'll need to do that in the next iteration, but it makes our code smaller and easier to focus on the core logic if we omit that in this step.

Let's start with the class members and constructor:

```typescript
// nestHttpApp/src/app.controller.ts
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

We should be familiar with the `serializer` and `deserializer` properties, and how they're initialized, from the server side.  The `fayeClient` holds our connection to the Faye broker and is the object we'll use to make Faye client API calls.  The constructor just sets this stuff up.

With that covered, we can look at the API our class exposes.  First, recall that the user-land call we need to support is:

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

We expose this feature, of course, with the `send()` method right at the top of the `faye-client.ts` file.

```typescript
// nestjs-faye-transporter/src/requestor/clients/faye-client.ts
  public send(pattern, data): Observable<any> {
    return new Observable(observer => {
      return this.handleRequest({ pattern, data }, observer);
    });
  }
```

As required, it returns an `Observable`. We use the normal Observable creation pattern, passing it a callback that handles emitting the values. In the callback, we use a factory method to implement the Observable subscription handler based on the pattern (e.g., `'/jobs-stream1'` ) and data (e.g., `duration`) in the request, since these are unknown until the call to `send()` is made.  This might sound a bit complicated, but let's break it down.

#### The Request Handler

The `handleRequest()` method is where the action is at.  If we work our way inside out, it will make sense.  A good place to start is to recall how we implemented this functionality in our native Faye `customerApp` way back in [Episode 1](https://dev.to/nestjs/build-a-custom-transporter-for-nestjs-microservices-dc6-temp-slug-6007254?preview=d3e07087758ff2aac037f47fb67ad5b465f45272f9c5c9385037816b139cf1ed089616c711ed6452184f7fb913aed70028a73e14fef8e3e41ee7c3fc#requestresponse).  Feel free to go have a quick look at that (here's a [link to the code](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/part4/customerApp/src/customer-app.ts#L20)). The quick summary for handling a request (a message expecting a response) is this:

1. Subscribe to the response channel (we know to use the pattern name followed by `'_res'`)
2. Send the request (we know to do this using the pattern name followed by `'_ack'`)
3. When the response comes in from the subscription (step 1), we *return it*
4. Recalling our **observable requirement**: *returning it* means *emitting the results as an observable stream*

This is really all we're doing in the `handleRequest()` method.  Let's take a look at it:

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

The `partialPacket` parameter has most of the elements of our outbound message.  It's been invoked from `send()` with an object like:
```json
{
  pattern: '/jobs-stream1/',
  data: 1
}
```

So we complete the formation of our outbound packet by adding that pesky `id` parameter (that we **still** haven't gotten to &#8212; I promise to address this in [Part 5](https://dev.to/nestjs/part-5-completing-the-client-component-hlh-temp-slug-2907984?preview=82c11163db963ca01d8d62d3a7b14843b422a6b28f46762d999bbe4b7035ad634d48bbbdd740e36376121aa673354ff5259f8b3028bceb931e800d9e)!).  After running our serializer, the packet is ready to send.

Let's skip the chunk beginning with the line `const subscriptionHandler = rawPacket => {` for a moment.

The next three chunks of code, respectively:
1. Perform our subscription, as discussed above. In Faye, the `subscribe()` call returns a promise that resolves when the Faye server returns an acknowledgement.
2. Upon that acknowledgement, we publish the serialized packet we built earlier.
3. Finally, we return a way to unsubscribe from the observable.  This is part of the Observable creation pattern.  Recall that ultimately, `handleRequest()` is a factory function returning the body of an observable subscription callback, so this is normal Observable creation.

Let's tackle the part we skipped a minute ago, where we build the subscription handler.  Here it is again:

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

Let's restate the purpose of this chunk of code.  We're building the callback that will be passed in our Faye client API `subscribe()` call to handle inbound messages.  As we handle each message, we are converting the sequence back into an observable stream.  We do that by taking the following steps for each inbound message from Faye:

1. Deserialize it and destructure it, leaving us with an `error`, `response` and `isDisposed` value for each inbound message
2. Determine if the observable stream has errored or is complete
3. Handle errors appropriately
4. Handle the next value in the stream until the `isDisposed` property is truthy, indicating the end of the stream coming from the server

As in any normal Observable, we handle these cases with the `observer.error()`, `observer.next()` and `observer.complete()` calls.

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

If all is working, you should get a coordinated set of logs in each of the terminal windows indicating that we have, indeed, successfully subscribed to the results of our remote request. :beer:!

### Limitations of Take 1

We knowingly deferred a few things to *Take 2* to keep focused on the core requirements and implementation of a client. We'll address them in [Part 5]().  But there's one **big** issue lurking.  Any guesses on what that might be? I've definitely foreshadowed this along the way (hint: repeatedly deferring the discussion of `id`).

OK, let's get right to it. Ask yourself this: since our client is embedded in an HTTP app that is responding to multiple incoming requests, many of which in turn are triggering remote requests, how does the transporter client keep track of all of the outstanding remote requests and route them back to the correct originating (HTTP) request?

Let's demonstrate the problem. You'll need one extra terminal window for this so we can launch two simultaneous requests.  You'll notice that in this branch, there is an extra route in `nestHttpApp`.  That route makes a new request (`'/race'`), which has a corresponding `@MessagePattern()` in `nestMicroservice`.  The point of these new features is to enable us to launch multiple requests that **overlap** in their execution.  Feel free to take a look at the code (these new routes/handlers are at the bottom of their respective Controller files), but all you really need to know is this: the `race` route in our `nestHttpApp` takes two parameters:
* a requestId
* a delay period (in seconds) specifying how long the server should wait before responding

For example, try making the following request (HTTPie format):

```bash
$ http get localhost:3000/race/1/10
```

This will return the following response, after 10 seconds:
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

With this in place, we can run two overlapping requests. Let's go ahead and run the following two requests.  You'll need to do this in two separate windows:

```bash
$ # run this in terminal 1, and be prepared to run the second one in terminal 2 a few seconds later
$ http get localhost:3000/race/1/10
$ # run this one in terminal 2, within a couple of seconds of running the top 1
$ http get localhost:3000/race/2/5
```

The results should (well, maybe not with all the foreshadowing :wink:) surprise you.  **Both** requests return the following result:

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

This is obviously incorrect for the first request.  The issue demonstrates what I'll refer to as the *[multiplexing](http://250bpm.com/blog:18) challenge*.  That is, we are multiplexing our requests through a single pair of channels.  When we issue request #1, it subscribes to a response on the `'/race_res'` channel. Request #2 does the same thing, and since request #2 finishes first, the request #1 subscription gets its expected result on the response channel.

Also, another nagging question... why do **both** subscribers get the result?  Hmmm... clearly something isn't working quite right here, and we'll have to work on this in our next (and final!) revision!

### What's Next

With these issues in mind, we're ready to build a final version of the Faye Custom Transporter client component, completing our development project.  In [Part 5](https://dev.to/nestjs/part-5-completing-the-client-component-hlh-temp-slug-2907984?preview=82c11163db963ca01d8d62d3a7b14843b422a6b28f46762d999bbe4b7035ad634d48bbbdd740e36376121aa673354ff5259f8b3028bceb931e800d9e) we cover:

* a
* b
* c

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
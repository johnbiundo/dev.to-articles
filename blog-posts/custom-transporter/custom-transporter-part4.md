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

In this article, we build the first iteration of the client component of the Faye Custom Transporter..

**Reminder**: Many of the concepts and terminology here are introduced and explained in [this article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3). That series serves as a good foundation for understanding the more advanced concepts covered in this series.

#### Get the Code

All of the code in these articles is available [here](https://github.com/johnbiundo/nestjs-faye-transporter-sample).  As always, these tutorials work best if you follow along with the code.  The [README](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md) covers all the steps you need to get the repository, build the apps, and follow along.  It's easy!  I strongly encourage you to do so.  Note that each article has a corresponding branch in the repository.  For example, this article (part 3), has a corresponding branch called `part3`. Read on, or [get more details here](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/README.md#part-4-initial-client-component) on using the repository.

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

In this animated sequence, our `nestMicroservice` *responder* app responds to a *request* message by emitting a stream of three results.  We saw in the last article how Nest converts that stream to a sequence of 3 messages to the broker on the response channel.  Our first meaningful task with the client component will be to consume those messages and convert them back into an observable in the `ClientProxy` layer so that our user-land controllers and services can use them as regular **RXjS** observables.

In this iteration (*Take 1*), we'll take a similar approach to our first iteration of the server.  We'll get a basic functioning client working so we can focus on its responsibilities and the main code path.  Later, we'll add more robustness.  As with server side, we'll ignore events for now (we won't support `ClientProxy#emit` in this version).

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

Once we instantiate a client like this, we can use it just like a `ClientProxy` object to make requests and emit events. Let's take a look at one such request.  Further down in the file, notice:

```typescript
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

The request above is the one we saw in the animation at the top of the article.  Feel free to take a look at the server-side code (`/nestMicroservice/src/app.controller.ts`) to get familiar with what's happening here.  You can also read the [deep dive on Observables](https://github.com/johnbiundo/nestjs-faye-transporter-sample/blob/master/observable-deepdive.md) for a thorough treatment (and a *treat* as well :smiley:).  For now, all we need to know is what we see in the animation above: when we make this request, we get a *stream of results* in the form of a sequence of messages.

Let's define our acceptance criteria for *Take 1* as follows.  Our `ClientFaye` object should let us send the `/jobs-stream1` request, and handle the response, converting the message stream back into an observable.

> **Note**: in [part5](https://dev.to/nestjs/part-5-completing-the-client-component-hlh-temp-slug-2907984?preview=82c11163db963ca01d8d62d3a7b14843b422a6b28f46762d999bbe4b7035ad634d48bbbdd740e36376121aa673354ff5259f8b3028bceb931e800d9e) we'll complete the implementation and have a fully functioning Faye Custom Transporter --xxx-- both client and server side.

### Take 1 Code Review



Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
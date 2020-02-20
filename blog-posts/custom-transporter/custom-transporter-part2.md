---
published: False
title: "Part 2: Basic Server Component"
description: "tags: nestjs, nest, faye, microservices, node.js"
series: "Advanced NestJS Microservices"
cover_image:
canonical_url:
---

*John is a member of the NestJS core team*

xxx longdash:
&#8212;

### Introduction

### Organizing the Code

Even though the scope of this tutorial is relatively small &#8212; at the end of the day we'll just be creating a few classes &#8212; we're going to be accumulating a number of assets along the way to test and exercise our code. It's time to think a little bit about organization.  First, let's take a look at what we'll end up with:

* Our Faye transporter is going to consist of some interfaces and constants, some serializers (discussed extensively in the [previous article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3)), and the main components: a **client** component (class) providing the `ClientProxy` subclass we'll use in our Nest requestor apps and a **server** component we'll use in our Nest responder apps.
* The native apps we've already written that let us explore the Faye API and give us some test drivers.
* A pair of regular Nest apps: a Nest requestor that is a simple Nest HTTP app called `nestHttpApp` and a Nest responder that is a **microservice**, called `nestMicroservice`.  These are the apps that will **use** our new Faye transporter.

Where to keep all of these assets?

The main thing to think about now is where to keep the Faye custom transporter components.  Since we'll want to easily re-use them across different Nest apps, the logical place is an NPM package.  To that end, when we work on any of those components, they'll go into that package.

#### The Custom Transporter Package

At this point, you should check out the `part2` branch of the repo that you cloned ([Read more here](xxx) for details).  Once you've checked out that branch, you'll notice a new folder called `nestjs-faye-transporter`.  This is organized as an NPM package (read more about creating NestJS-friendly NPM packages in my article [Publishing NestJS Packages with npm](https://dev.to/nestjs/publishing-nestjs-packages-with-npm-21fm), including how the `tsconfig.json` file, `package.json` scripts, and file organization all interact to create a reusable NPM package).

Here's a run-down on the contents of that package, and how we'll use each:

1. The scripts (`package.json`) let you build (`npm run build` or `npm run build:watch`) and, if you want, publish (`npm publish`) the package.
2. Our workflow, in dev, will be to build the package and then use `npm link` to install it in our Nest apps.  More on this soon.
3. The code is organized in a directory structure as follows:
     * Code common to our **client** and **server** is in top level files or folders, like `src/external` (holds things like interfaces to the Faye client library which are *external* to the Nest environment), `src/interfaces` (holds interfaces used within the Nest constructs), and `src/constants`.
     * All code defining the **client** is in the `src/requestor` folder. Ultimately, this will consist of a sub-class of `ClientProxy` from the `@nestjs/microservices` package &#8212; the class which provides things like `client.send(...)` and `client.emit()`) &#8212; and a variety of related classes.
     * All code defining the **server** is in the `src/responder` folder.  Ultimately, this will consist of a sub-class of `Server` from the `@nestjs/microservices` package &#8212; the class which implements a transporter strategy &#8212; and a variety of related classes.

### First Iteration (Take 1) of the Server Component

Let's get started building the "server" side of the equation &#8212; the part of the custom transporter that you'll use to build Nest responders (see [the previous article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) for more on this terminology, but think of a Nest responder as a Nest application that listens for inbound requests over a specific transport, like Faye).

Again, for this step of the tutorial, we should be on the `part2` branch.

#### Using a Custom Transporter Strategy

Before we start writing transporter code, let's first take a look at how we're going to use it.  For that purpose, we'll have a simple nest responder microservice app &#8212; the by-now-familiar `nestMicroservice` app.  The code for this app is in the top-level `nestMicroservice` folder.

Open the `main.ts` file.  Notice the structure of the `createMicroservice` call:

```ts
const app = await NestFactory.createMicroservice(AppModule, {
  strategy: new ServerFaye({
    url: 'http://localhost:8000/faye',
    retry: 5,
    timeout: 120,
    serializer: new OutboundResponseIdentitySerializer(),
    deserializer: new InboundMessageIdentityDeserializer(),
  }),
});
```

This is very similar to the structure used for any built-in Nest transporter.  For example, with MQTT it looks like:


```ts
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    host: 'localhost',
    port: 1883,
    serializer: new OutboundResponseIdentitySerializer(),
    deserializer: new InboundMessageIdentityDeserializer(),
  },
});
```

It should be clear what we're doing: instead of passing the normal [transporter options object](https://docs.nestjs.com/microservices/basics#getting-started) with `transport` and `options` properties, we pass a single property, `strategy`, whose value is an **instance** of our custom transporter class.

#### Take 1 Requirements

Alright, we're finally ready to look at our `ServerFaye` class &#8212; the one we'll instantiate and pass as the value of that `strategy` property above.  We'll take this in a few steps.

This iteration ("Take 1") is going to be the absolute bare-bones class needed to implement the Faye transporter.  We're going to make a number of simplifications, so that we can focus on the core flow.  Our first cut will:

- minimize error handling
- not rely on some nice features of the framework that make our code really robust
- not handle events (e.g., inbound messages coming from `client.emit(...)`)
- not really be type safe (we omit a bunch of typing to declutter the code)

To state it in more direct terms, our requirement for this cut is to simply be able to respond to a well-formed an inbound **request** (in the sense of a request from a **request-response** style message).  We'll test this requirement by replacing our native `customerService` responder app from the last article with our `nestMicroservices` app running our new custom transporter, and sending our `nestMicroservice` some requests from our native `customerApp`.

**Note:** In [part 3](), we'll complete the implementation and have a fully functioning Faye Custom Transporter (server component).  At that point, you'll also have all of the concepts in place to write your own custom transporter server component, as well as the ability to look inside the built-in transporters (like [the MQTT transporter server](https://github.com/nestjs/nest/blob/master/packages/microservices/server/server-mqtt.ts)) and understand what's going on.  That will prepare you for even more adventures, like customizing the Nest built-in transporters to add features&#8212; the subject of my next NestJS microservice tutorial (already underway, and coming very soon)!

#### Take 1 Code Review

Let's dive into the code.  Open the file `nestjs-faye-transporter/src/responder/transporters/server-faye.ts`.

An important concept is that this class extends `Server` (view this parent class here: [@nestjs/microservices/server/server.ts](https://github.com/nestjs/nest/blob/master/packages/microservices/server/server.ts)). We won't walk through all the details of the `ServerFaye` class, but the main things to know are:

1) It inherits some properties and methods from the built-in `Server` class
2) This class fits into the Nest lifecycle. What that means is that when the application bootstraps, our custom `ServerFaye` class is automatically instantiated, its inherited properties are populated with data, and the framework calls the entry point of our class.  That entry point is the `listen()` method.

So let's start with `listen()`. Its job is to make a connection to the broker, and then run `start()`, which is where the fun begins.

The interesting thing in `start()` is the call to `this.bindHandlers()`.  Take a look at that method:

```ts
  public bindHandlers() {
    /**
     * messageHandlers is populated by the Framework (on the `Server` superclass)
     *
     * It's a map of `pattern` -> `handler` key/value pairs
     * `handler` is the handler function in the user's controller class, decorated
     * by `@MessageHandler()` or `@EventHandler`, along with an additional boolean
     * property indicating it's Nest pattern type: event or message (i.e.,
     * request/response)
     */
    this.messageHandlers.forEach((handler, pattern) => {
      // only handling `@MessagePattern()`s for now
      if (!handler.isEventHandler) {
        this.fayeClient.subscribe(
          `${pattern}_ack`,
          this.getMessageHandler(pattern, handler),
        );
      }
    });
  }

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

The comments should help with understanding what's happening, but at a high level, the concept is pretty clear:
1. Iterate over all of our "message handlers" (these are the user-land code like handlers decorated with `@MessagePattern()`; they're made available to us by the framework which performs introspection using the `Reflect` API  during the bootstrap process).\*
2. For each one, subscribe to the inbound channel (the `_ack` form of the topic).  Remember, a Faye client's `subscribe()` call registers a callback handler to be invoked whenever the Faye client library receives an inbound message matching this topic.
3. The subscription handler in step 2, when invoked, runs the *actual* pattern handler (the user-land handler method decorated with something like `@MessagePattern('get-customer')`), and *returns* the result it gets from that method.  Of course in our scenario, returning means publishing a reply on the outbound channel.
4. Along the way, we run our deserializer on the inbound message, our serializer on the outbound response, and we package up the data produced by the user's pattern handler in an appropriately shaped standard Nest transporter message object.

\*We omit event handlers (those registered with `@EventPattern(...)`) for now.  We'll handle these in Take 2 of our server component, in the next article.

If all this looks somewhat familiar, it's because we're basically following the same approach we used in the native `customerService` app.

### Acceptance Testing

We should be ready to test out our code.  We need to do a tiny bit of setup first.  This is going to run best if you have four (or even five) separate terminals open so you can watch things unfold across the whole process as the various pieces communicate.

Now's a good time to mention a couple of things about the development setup:
1. I run this kind of stuff on an Ubuntu machine.  There's **nothing** platform-specific anywhere in the code, **but**, I find running builds, starting and stopping processes, running multiple terminal sessions, etc., to be much smoother on Linux.  You can run this wherever you want, but if you're not on Linux, you may have to make slight adjustments to your `package.json`, or other actual build steps.
2. Since you kind of **need** to run multiple terminal sessions to see the full impact, I strongly recommend [Tmux](xxx).  You can start multiple terminal programs, or use tabs if you prefer, but if you want what I think is the **best** DX for this kind of work, checkout Tmux.  I covered some of my recommendations [in the last article series](https://github.com/johnbiundo/nest-nats-sample#pro-tip-use-tmux-optional).

In the following steps, I'll reference these (logical) terminals as:
* terminal 1: run the Faye broker here
* terminal 2: run builds of the transporter server code we're working on here
* terminal 3: run the customerApp and HttPie commands here (you can also use Postman, or Curl to issue HTTP requests, of course)
* terminal 4: run the `nestMicroservice` testing application (this is the plain old Nest app that is **using** our new `ServerFaye` custom transporter)
* terminal 5: run the `nestHttpApp` here

#### Primary Acceptance Test

Going back to our requirements, our main acceptance test is simple: send a well-formed message from our native `customerApp` (acting as a requestor) to our `nestMicroservice` (acting as a responder), and get back a proper response.

((Diagram))

##### Using `npm link` in Our Development Environment

Before we can run the test, we need to cover one more preliminary.  Ask yourself this: how does our `nestMicroservice` app know that we want to use the code in our `nestjs-faye-transporter` package?  The answer is pretty simple (though it belies the awesomeness of NPM!).

We're going to use the `npm link` command (read details [here]()).  There are two simple steps:

1. In terminal 2, make sure you're in the folder `nestjs-faye-transporter` (our NPM package folder).  Then run `npm link` at the OS level.  You should see output like this:

    ```bash
    xxx blah
    ```

2. In terminal 4, run `npm link @faye-tut/nestjs-faye-transporter` at the OS level.  This is the NPM package name (found in `nestjs-faye-transporte/package.json`) for our custom transporter.  You should see output like this:

    ```bash
    xxx blah
    ```

At this point, our `nestMicroservice` testing app is live-linked to the custom transporter code in the `@faye-tut/nestjs-faye-transporter` NPM package we're working on.

##### Running the Test

We're ready to rock and roll!

In terminal 1, make sure the Faye broker is running. Make sure you're in the `faye-server` directory, then run `npm run start`.  You should see something like this:

```bash
xxx blah
```

In terminal 2, start the `nestMicroservice`.  Make sure you're in the `nestMicroservice` directory, then run `npm run start:dev`.  You should see the usual Nest startup logs, followed by the message `Microservice is listening...`.

In terminal 3, we'll run the native `customerApp` to make the request.  Earlier, we used this to send requests to the native `customerService` app.  Now we're sending those messages (via the Faye broker, of course) to the `nestMicroservice`.  Make sure you're in the `customerApp` directory, then...

... keep your eyes on all 3 terminal windows, and...

run `npm run get-customers`.

If all went well, you should see a flurry of log messages.  Do take the time to look at them and make sure you can follow the flow of what's happening.  In terminal 3, we should get a nice response that looks like this:

```bash
$ ► npm run get-customers

> faye-customer-app@1.0.0 get-customers /home/john/code/nest-micro/nestjs-faye/customerApp
> node dist/customer-app get

Faye customer app starts...
===========================
<== Sending 'get-customers' request with payload:
{"pattern":"/get-customers","data":{"requestId":0,"requestDelay":0},"id":"aa0ac7d9-35fa-4b44-839d-6da078583250"}

==> Receiving 'get-customers' reply (request: 0):
{
  "customers": [
    {
      "id": 1,
      "name": "nestjs.com"
    }
  ],
  "requestId": 0,
  "delay": 0
}
```

Hooray! We're done right?  :beer:?

Not so fast... did you forget this is a five-part series? :smiley:

Read on to see where we've still got work to do.

### Understanding the Limitations of Take 1

We already know we have to clean up a few things, like adding event handling, adding TypeScript types, and plugging in more cleanly to the framework.  But the biggest limitations are these:

1. Things work fine if our handlers return plain objects.  For example, we currently return an object in our `nestMicroservices` `getCustomers()` handler (last line below):

    ```typescript
    // nestMicroservice/src/app.controller.ts
    @MessagePattern('/get-customers')
    async getCustomers(data: any): Promise<any> {
      const delay = (data.requestDelay && data.requestDelay * 1000) || 0;
      const requestId = data.requestId || 0;
      console.log(`delay: ${delay}, requestId: ${requestId}`);

      function sleep() {
        return new Promise(resolve => {
          setTimeout(() => {
            resolve();
          }, delay);
        });
      }

      await sleep();
      const customers =
        data && data.customerId
          ? customerList.filter(cust => cust.id === parseInt(data.customerId, 10))
          : customerList;
      return { customers, requestId, delay };
    }
    ```

    But what happens if our handler returns an observable? The framework enables this for all built-in transporters, so we should too.  Let's test this.  Replace that last line with:

    ```typescript
    return of({customers, requestId, delay});
    ```

    You'll also have to add this line to the top of the file:
    ```typescript
    import { of } from 'rxjs';
    ```

    This causes our handler to return an **observable** &#8212; a stream (containing only a single value in our case, but still, a stream) of values.

    If you make this change, then reissue the `get-customers` message (run `npm runget-customers` in terminal 3), you'll get a rather ugly failure in the `nestMicroservice` window.  We aren't handling this case, which, again, is expected of any Nest microservice transporter.

2. So far, we've only issued a single outstanding request to our responder app at a time.  What happens if, as in the real world, we start hitting it with multiple, overlapping requests?  Let's test this.</br></br>

    If you ran the previous test (returning `of({customers, requestId, delay});` from `nestMicroservice/src/app.controller.ts`, undo that now: restore the final line to `return { customers, requestId, delay };` ).</br></br>

    Now, with the `nestMicroservice` app running in terminal 4, run the following command in terminal 3 (which should be in the `customerApp` directory):

    ```bash
    npm run race
    ```



### Summary
Here's what it looks like:

![Native App Demo](./assets/native-app-demo.gif 'Native App Demo')
<figcaption><a name="screen-capture-2"></a>Screen Capture 2: Native App Demo</figcaption>

### What's Next

In [Part 3](xxx),

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
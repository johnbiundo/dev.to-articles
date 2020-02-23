---
published: False
title: "Part 2: Basic Server Component"
description: "tags: nestjs, nest, faye, microservices, node.js"
series: "Advanced NestJS Microservices"
cover_image: "https://dev-to-uploads.s3.amazonaws.com/i/0pg4mzdjilg9qfpn819w.gif"
canonical_url:
---

*John is a member of the NestJS core team*

xxx longdash:
&#8212;

### Introduction

**Reminder**: Many of the concepts and terminology here are introduced and explained in [this article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3). That series serves as a good foundation for understanding the more advanced concepts covered in this series.

** get the code **

** build the apps **

### Organizing the Code

Even though the scope of this tutorial is relatively small &#8212; at the end of the day we'll just be creating a few classes &#8212; we're going to be accumulating a number of assets along the way to test and exercise our code. It's time to think a little bit about code organization.  First, let's take a look at what we'll end up with:

* Our Faye transporter is going to consist of some interfaces and constants, some serializers/deserializers (discussed extensively in the [previous article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3)), and the main components: a **client** component (class) providing the `ClientProxy` subclass we can use in any *Nest requestor* apps and a **server** component (class) we can use in any *Nest responder* apps.
* The native apps we wrote in the [previous article]() that let us explore the Faye API and give us some test drivers.
* A pair of regular Nest apps that will exercise our transporter code: a *Nest requestor* that is a simple Nest HTTP app called `nestHttpApp` and a *Nest responder* that is a classic Nest **microservice**, called `nestMicroservice`.  (**Note**: see [the previous article series](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) for more on this terminology, but think of a *Nest Requestor* as an app that makes remote requests over some (non-HTTP) transport, like Faye, and a Nest responder as a Nest application that listens for inbound requests over that transport, like Faye).

Where to keep all of these assets?

The main thing to think about now is where to keep the Faye custom transporter components.  Since we'll want to easily re-use them across different Nest apps, the logical place is an NPM package.  To that end, when we work on any of those components, they'll go into that package.

#### The Custom Transporter Package

At this point, you should `git checkout` the `part2` branch of the repo that you cloned ([Read more here](xxx) for details).  Once you've checked out that branch, you'll notice a new folder called `nestjs-faye-transporter`.  This is organized as an NPM package (you can learn more about creating NestJS-friendly NPM packages in my article [Publishing NestJS Packages with npm](https://dev.to/nestjs/publishing-nestjs-packages-with-npm-21fm), including how the `tsconfig.json` file, `package.json` scripts and properties, and file organization all interact to create a reusable NPM package).

Here's a run-down on the contents of the `nestjs-faye-transporte` package, and how we'll use each:

1. The scripts (`package.json`) let you build (`npm run build` or `npm run build:watch`) and, if you want, publish (`npm publish`) the package.
2. Our workflow, in dev, will be to build the package and then use `npm link` to install it in our Nest apps.  More on this soon.
3. The code is organized in a directory structure as follows:
     * Code common to our **client** and **server** is in top level files or folders, like `src/external` (holds things like interfaces to the Faye client library which are *external* to the Nest environment), `src/interfaces` (holds interfaces used within the Nest constructs), and `src/constants`.
     * All code defining the **client** is in the `src/requestor` folder. Ultimately, this will consist of a sub-class of `ClientProxy` from the `@nestjs/microservices` package &#8212; the class which provides methods like `client.send(...)` and `client.emit(...)`) &#8212; and a variety of related classes.
     * All code defining the **server** is in the `src/responder` folder.  Ultimately, this will consist of a sub-class of `Server` from the `@nestjs/microservices` package &#8212; the class which implements a transporter strategy &#8212; and a variety of related classes.

### First Iteration (Take 1) of the Server Component

Let's get started building the "server" side of the equation &#8212; the part of the custom transporter that you'll use to build *Nest responders*.

Again, for this step of the tutorial, we should be on the `part2` branch of the repository.

#### Using a Custom Transporter Strategy

Before we start writing transporter code, let's first take a look at how we're going to use it.  For that purpose, let's look at the `nestMicroservice` app.  This app has been added in this branch. The code for it is in the top-level `nestMicroservice` folder.

##### Instantiating the Custom Transporter

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

It should be clear what we're doing: instead of passing the normal [transporter options object](https://docs.nestjs.com/microservices/basics#getting-started) with `transport` and `options` properties, we pass a single property, `strategy`, whose value is an **instance** of our custom transporter class.  This class takes a single options object in its constructor, pretty much mirroring the options you pass to the `options` property with the built-in transporters.

##### Using the Custom Transporter

Let's briefly review how our `nestMicroservice` app works.  It's only real job is to take in a `'/get-customers'` message and return a list of customers.  The action is in the `nestMicroservices/src/app.controller.ts` file:

```typescript
@Controller()
export class AppController {
  logger = new Logger('AppController');

  /**
   * Register a message handler for 'get-customers' requests
   */
  @MessagePattern('/get-customers')
  async getCustomers(data: any): Promise<any> {
    const customers =
      data && data.customerId
        ? customerList.filter(cust => cust.id === parseInt(data.customerId, 10))
        : customerList;
    return { customers };
  }
}
```

All it really does is check for the existence of a customerId, and if it exists, uses it to filter the customer list it returns.

#### Take 1 Requirements

Alright, we're finally ready to look at our `ServerFaye` class &#8212; the one we'll instantiate and pass as the value of that `strategy` property above.  We'll take this in a couple of steps.

This iteration ("Take 1") is going to be the absolute bare-bones class needed to implement the Faye transporter.  We're going to make a number of simplifications, so that we can focus on the core flow.  Our first cut will:

- minimize error handling
- not rely on some nice features of the framework that make our code really robust
- not handle events (e.g., inbound messages coming from `client.emit(...)`)
- not really be type safe (we omit a bunch of typing to declutter the code)

To state it in more direct terms, our requirement for this cut is to simply be able to respond to a well-formed inbound **request** (in the sense of a request from a **request-response** style message).  We'll test this requirement by replacing our native `customerService` responder app from the last article with our `nestMicroservices` app running our new custom transporter, and sending it some requests from our native `customerApp`.

> **Note:** In [part 3](), we'll complete the implementation and have a fully functioning Faye Custom Transporter (server component).  At that point, you'll also have all of the concepts in place to write your own custom transporter server component, as well as the ability to look inside the built-in transporters (like [the MQTT transporter server](https://github.com/nestjs/nest/blob/master/packages/microservices/server/server-mqtt.ts)) and understand what's going on.  That will prepare you for even more adventures, like customizing the Nest built-in transporters to add features&#8212; the subject of my next NestJS microservice tutorial (already underway, and coming very soon :boom:)!

#### Take 1 Code Review

Let's dive into the code.  Open the file `nestjs-faye-transporter/src/responder/transporters/server-faye.ts`.

An important concept is that this class extends `Server` (view this parent class here: [@nestjs/microservices/server/server.ts](https://github.com/nestjs/nest/blob/master/packages/microservices/server/server.ts)). We won't walk through all the details of the `ServerFaye` class, but the main things to know are:

1) It inherits some properties and methods from the built-in `Server` class
2) The class is *managed for us* as part of the Nest lifecycle. This means that when the application bootstraps, our custom `ServerFaye` class is automatically instantiated, its inherited properties are populated with data, and the framework calls the entry point of our class.  That entry point is the `listen()` method.

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

The comments should help with understanding what's happening, but at a high level, the strategy should be pretty clear:
1. Iterate over all of our "message handlers" (these are the user-land Controller methods decorated with `@MessagePattern()`; a list of them is made available to us  (`this.messageHandlers`) by the framework which performs introspection using the `Reflect` API  during the bootstrap process).\*
2. For each one, subscribe to the inbound channel (the `_ack` form of the topic).  Remember, a Faye client's `subscribe()` call registers a callback handler to be invoked whenever the Faye client library receives an inbound message matching this topic.  So this is the step where we map *patterns* to *handlers*. In short, this is our "router".
3. The subscription handler in step 2, when invoked, runs the *actual* user supplied handler (the user-land handler method decorated with something like `@MessagePattern('get-customer')`), and *returns* the result it gets from that method.  When we say "returns", we mean **publishes a reply** on the outbound channel (the `_res` form of the topic).
4. Along the way, we run our deserializer on the inbound message, our serializer on the outbound response, and we package up the data produced by the user's pattern handler in an appropriately shaped standard Nest transporter message object.

\*We are omitting *event handlers* (methods decorated with `@EventPattern(...)`) for now.  We'll handle these in Take 2 of our server component, in the next article.

If all this looks somewhat familiar, it's because we're basically following the same approach we used in the native `customerService` app.

### Acceptance Testing

We should be ready to test out our code.  We need to do a tiny bit of setup first.  This is going to run best if you have four separate terminals open so you can watch things unfold across the whole process as the various pieces communicate.

Now's a good time to mention a couple of things about the development setup:
1. I run this kind of stuff on an Ubuntu machine.  There's **nothing** platform-specific anywhere in the code, **but**, I find running builds, starting and stopping processes, running multiple terminal sessions, etc., to be much smoother on Linux.  You can run this wherever you want, but if you're not on Linux, you may have to make slight adjustments to your `package.json`, or other actual build steps.
2. Since you kind of **need** to run multiple terminal sessions to see the full impact, I strongly recommend [Tmux](xxx).  You can start multiple terminal programs, or use tabs if you prefer, but if you want what I think is the **best** DX for this kind of work, checkout Tmux.  I covered some of my recommendations [in the last article series](https://github.com/johnbiundo/nest-nats-sample#pro-tip-use-tmux-optional).

In the following steps, I'll reference these (logical) terminals as:
* terminal 1: run the Faye broker here
* terminal 2: run live builds (`npm build:watch`) of the transporter server code we're working on here
* terminal 3: run the "requestor code".  This is usually the customerApp; in the future, we'll also interact with `nestHttpApp` (making **it** the requestor) using HttPie commands from the OS prompt (you can also use Postman, or Curl to issue HTTP requests, of course). We can use one terminal for this since we don't typically run both the `customerApp` and the `nestHttpApp` at the same time
* terminal 4: run the `nestMicroservice` Nest responder application (this is the plain old Nest microservice app that will be **using** our new `ServerFaye` custom transporter)

#### Primary Acceptance Test

Going back to our requirements, our main acceptance test is simple: send a well-formed message from our native `customerApp` (acting as a requestor) to our `nestMicroservice` (acting as a responder), and get back a proper response.

![Faye Transporter Acceptance Test](./assets/acceptance-test.png 'Faye Transporter Acceptance Test')
<figcaption><a name="figure1"></a>Figure 1: Faye Transporter Acceptance Test</figcaption>

##### Using `npm link` in Our Development Environment

Before we can run the test, we need to cover one more preliminary.  Ask yourself this: how does our `nestMicroservice` app know how to find the code in our `nestjs-faye-transporter` package?  The answer is pretty simple (though it belies the awesomeness of NPM!).

We're going to use the `npm link` command ([read details here](https://medium.com/dailyjs/how-to-use-npm-link-7375b6219557), but the following all you need to know for this tutorial).  There are two simple steps:

1. In terminal 2, make sure you're in the folder `nestjs-faye-transporter` (our NPM package folder).  Then run `npm link` at the OS level.  You should see output like this:

    ```bash
    $ npm link

    @faye-tut/nestjs-faye-transporter@1.0.1 prepare /home/john/code/nest-micro/nestjs-faye/nestjs-faye-transporter
    npm run build

    @faye-tut/nestjs-faye-transporter@1.0.1 build /home/john/code/nest-micro/nestjs-faye/nestjs-faye-transporter
    tsc

     (... some lines omitted ...)

    /home/john/.nvm/versions/node/v10.18.1/lib/node_modules/@faye-tut/nestjs-faye-transporter ->
    /home/john/code/nest-micro/nestjs-faye/nestjs-faye-transporter
    ```

2. In terminal 4, make sure you're in the folder `nestMicroservice`.  Then run `npm link @faye-tut/nestjs-faye-transporter` at the OS level.  This command references the NPM package name (found in `nestjs-faye-transporter/package.json`) for our custom transporter.  You should see output like this:

    > $ npm link @faye-tut/nestjs-faye-transporter
    > /home/john/code/nest-micro/nestjs-faye/nestMicroservice/node_modules/@faye-tut/nestjs-faye-transporter ->
    > /home/john/code/nest-micro/nestjs-faye/nestjs-faye-transporter


At this point, our `nestMicroservice` testing app is live-linked to the custom transporter code in the `@faye-tut/nestjs-faye-transporter` NPM package we're working on.

##### Running the Test

We're ready to rock and roll :musical_note:!

In terminal 1, make sure the Faye broker is running. First make sure you're in the `faye-server` directory, then run `npm run start`.  You should see something like this:

> $ npm run start
> \> simple-fay-eserver@1.0.0 start /home/john/code/nest-micro/nestjs-faye/faye-server
> \> node server.js
>
> listening on http://localhost:8000/faye
> \========================================

In terminal 2, start the `nestMicroservice`.  First make sure you're in the `nestMicroservice` directory, then run `npm run start:dev`.  You should see the usual Nest startup logs, followed by the message `Microservice is listening...`.

In terminal 3, we'll run the native `customerApp` to make the request.  Earlier, we used this to send requests to the native `customerService` app.  Now we're sending those messages (via the Faye broker, of course) to the `nestMicroservice`.  Make sure you're in the `customerApp` directory, then...

... keep your eyes on all 3 terminal windows, and...

run `npm run get-customers`.

If all went well, you should see a flurry of log messages.  I strongly encourage you to take the time to look at them and make sure you can follow the flow of what's happening.  In terminal 3, we should get a nice response that looks like this:

```bash
$ npm run get-customers

> faye-customer-app@1.0.0 get-customers /home/john/code/nest-micro/nestjs-faye/customerApp
> node dist/customer-app get

Faye customer app starts...
===========================
<== Sending 'get-customers' request with payload:
{"pattern":"/get-customers","data":{},"id":"415fdcef-88e8-42e7-a464-ebdb3b10ff57"}

==> Receiving 'get-customers' reply:
{
  "customers": [
    {
      "id": 1,
      "name": "nestjs.com"
    }
  ]
}
```

Hooray! We're done right?  :beer:?

Not so fast... did you forget this is a five-part series :smiley:? Read on to see where we've still got work to do.

### Understanding the Limitations of Take 1

We already know we have to clean up a few things, like adding **event handling** (e.g., handlers decorated with `@EventPattern(...)`), adding TypeScript types, and plugging in more cleanly to the framework.  But the biggest limitation of our Take 1 implementation is its (lack of) handling of [RxJS observables](http://reactivex.io/).  To fully appreciate this, we'll have to take a bit deeper dive into the overall flow and handling of requests through the Nest system.

We'll explore this in greater detail in the next article, but let's start with a picture.  The following animation shows the path a hypothetical inbound HTTP request would take through our application. Inner boxes with a red background are part of the Nest infrastructure. User supplied code lives in the "user land" space with a white background.  Controllers are in light blue, and "Services" are in yellow.

![Nest Request Handling](./assets/transporter-request.gif 'Nest Request Handling')
<figcaption><a name="Nest Request Handling"></a>Figure 1: Nest Request Handling</figcaption>

An inbound HTTP request kicks off the following sequence of events.  Bolded words represent Nest system responsibilities.  Underlined words represent user code. There's probably nothing terribly surprising going on here, but let's just briefly walk through it.
1. The request is **routed** to a <u>route handling method</u> (like `getCustomers` in our controller).
2. The <u>route handling method</u> can either make a remote request directly, or <u>call a service that makes a remote request</u>.
3. The remote request is handled by **the broker client library** and sent to the broker.
4. The remote request is received by **the broker client library** in the `nestMicroservice` app.
5. The request is **routed** to the correct handler (e.g., a method decorated with `@MessagePattern('get-customers')`) based on matching the pattern in the request with the pattern in the method decorator.
6. The <u>request handling method</u> may make a call to other services (which in turn, could make their own internal calls or remote calls).  Let's say it does make such a call, to the `custService.getCustomers()` method, and that method has a signature like this:

    ```typescript
    getCustomers(id: integer): Observable<Customer>
    ```

Once the `getCustomers` method returns, we start the *return trip*, where things get more interesting. This is mainly because Nest is very **observable-aware**.

![Nest Response Handling](./assets/transporter-response2.gif 'Nest Response Handling')
<figcaption><a name="Nest Response Handling"></a>Figure 2: Nest Response Handling</figcaption>

In this sequence, I'll introduce the role of what I'm informally calling the "Mapper" (there's no such official term or single component inside Nest called a Mapper).  Conceptually, it's the part(s) of the system that handle(s) dealing with Observables.

> In the next article, we'll go through a few use cases for **why observables are so cool, why they're a perfect fit in this flow, AND how they're actually really easy to use**.  I know this diagram doesn't make it seem that way, but hey, we're building our own transporter (my inner [trekky](http://sfi.org/) can't help but giggle over that :rocket:).  The beauty of it is that once we handle this case properly &#8212; and the framework will make this easy as we'll see in the next chapter &#8212; everything we might want to do with Observables (and their potential is just, well, *enormous*) **just works**.

Here's the walk through of the return trip flow:

1. <u>Our handler</u> in `nestMicroservice` (on its own, perhaps, or by getting a return value from a service it calls) returns the result. In terms of our example, it's returning a list of customers.
2. Now, if the response is a "plain" value (JavaScript primitive, array or object), there's not much work to be done other than send it back to the broker. But, if the response is a Promise or Observable, **Nest steps in and makes this extremely easy** for everything downstream to work with.  That's the job of the **mapper**.
3. Once the response is prepared from `nestMicroservice`, it's **delivered to the Faye broker** via the broker client library.
4. The broker takes care of **publishing the response**.
5. On the `nestHttpApp` side, the broker client library (which has previously subscribed to the broker on the *response channel* &#8212; though we haven't coded that part yet), **receives the response message**.
6. The `ClientProxy` class (again, we haven't built this yet) takes care of **routing** (and **mapping**, if it's dealing with non-primitive responses).
7. This routes the response to the <u>originating service</u>, then back to the <u>originating controller</u>.
8. Finally, the Nest HttpAdapter software again, if needed, **maps responses** to a suitable form for HTTP transport.  For example, if the response is an observable or a promise, it **converts it to an appropriate form** for return over HTTP.

So what exactly is the issue?

As you might expect from the above, things work fine if our handlers return plain objects.  For example, we currently return an object in our `nestMicroservices` `getCustomers()` handler (last line below):

```typescript
// nestMicroservice/src/app.controller.ts
@MessagePattern('/get-customers')
async getCustomers(data: any): Promise<any> {
  const customers =
    data && data.customerId
      ? customerList.filter(cust => cust.id === parseInt(data.customerId, 10))
      : customerList;
  return { customers };
}
```

But what happens if our handler returns an observable? The framework enables this for all built-in transporters, so we should handle it too.  Let's test this.  Replace that last line in `app.controller.ts` with:

```typescript
return of({customers, requestId, delay});
```

You'll also have to add this line to the top of the file:
```typescript
import { of } from 'rxjs';
```

This causes our handler to return an **observable** &#8212; a stream (containing only a single value in our case, but still, a stream) of values.

If you make this change, then re-issue the `get-customers` message (run `npm run get-customers` in terminal 3), you'll get a rather ugly failure in the `nestMicroservice` window.  We aren't handling this case, which, again, is expected of any Nest microservice transporter.

### What's Next

With these issues in mind, we're ready to step up our game and make the `ServerFaye` custom transporter much more robust.  We'll tackle that in the next article.

In [Part 3](xxx), we cover:
* A
* B
* C

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
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

=====================
Probably Part 2, but not clear yet
=====================

### Organizing the Code

Even though the scope of this tutorial is relatively small &#8212; at the end of the day we'll just be creating a few classes &#8212; we're going to be accumulating a number of assets along the way to test and exercise our code. It's time to think a little bit about organization.  First, let's take a look at what we'll end up with:

* Our Faye transporter is going to consist of some interfaces and constants, some serializers (discussed extensively in the [previous article series]()), and the main components: a **client** component (class) providing the `ClientProxy` subclass we'll use in our Nest requestor apps and a **server** component we'll use in our Nest responder apps.
* The native apps we've already written that let us explore the Faye API and give us some test drivers.
* A pair of regular Nest apps: a Nest requestor that is a simple Nest HTTP app called `nestHttpApp` and a Nest responder that is a **microservice**, called `nestMicroservice`.  These are the apps that will **use** our new Faye transporter.

Where to keep all of these assets?

The main thing to think about now is where to keep the Faye custom transporter components.  Since we'll want to easily re-use them across different Nest apps, the logical place is an NPM package.  To that end, from this point forward when we work on any of those components, they'll go into that package.

#### The Custom Transporter Package

At this point, you should check out the `part2` branch of the repo that you cloned.  [Read more here](xxx) for details.  Once you've checked out that branch, you'll notice a new folder called `nestjs-faye-transporter`.  This is organized as an NPM package (read more about creating NestJS-friendly NPM packages in my article [Publishing NestJS Packages with npm](https://dev.to/nestjs/publishing-nestjs-packages-with-npm-21fm), including how the `tsconfig.json` file, `package.json` scripts, and file organization all cooperate to create a reusable NPM package).

Here's a run-down on the contents of that package, and how we'll use it.

1. The scripts (`package.json`) let you build (`npm run build`), run (`npm run build`) and, if you want, publish (`npm publish`) the package.
2. Our workflow, in dev, will be to build the package and then use `npm link` to install it in our Nest apps.  More on this soon.
3. The code is organized in a directory structure as follows:
     * Code common to our **client** and **server** is in top level files or folders, like `src/external` (holds things like interfaces to the Faye client library which are *external* to the Nest environment), `src/interfaces` (holds interfaces used within the Nest constructs), and `src/constants`.
     * All code defining the **client** is in the `src/requestor` folder. Ultimately, this will consist of a sub-class of `ClientProxy` from the `@nestjs/microservices` package &#8212; the class which provides things like `client.send(...)` and `client.emit()`) &#8212; and a variety of related classes.
     * All code defining the **server** is in the `src/responder` folder.  Ultimately, this will consist of a sub-class of `Server` from the `@nestjs/microservices` package &#8212; the class which implements a transporter strategy &#8212; and a variety of related classes.

### First Iteration of the Server Component

Let's get started building the "server" side of the equation -- the part of the custom transporter that you'll use to build Nest responders (see [xxx]() for more on this terminology, but think of a Nest responder as a Nest application that listens for inbound requests over a specific transport, like Faye).

Again, for this step of the tutorial, we should be on the `part2` branch.

#### Using a Custom Transporter Strategy

Before we start writing transporter code, let's first take a look at how we're going to use it.  We'll plug it into the by-now-familiar `nestMicroservice` app.  The code for this app is in the top-level `nestMicroservice` folder.

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

#### Take One

Alright, we're finally ready to look at our `ServerFaye` class &#8212; the one we'll instantiate and pass as that `strategy` property above.  We'll take this in a few steps.  Start by opening the file `nestjs-faye-transporter/src/responder/transporters/server-faye.ts`.

This is going to be the absolute bare-bones class needed to implement the Faye transporter.  We're going to make a number of simplifications, so that we can focus on the core flow.  Our first cut will:

- minimize error handling
- not rely on some nice features of the framework that make our code really robust
- not handle events (e.g., inbound messages coming from `client.emit(...)`)
- not really be type safe (we omit a bunch of typing to declutter the code)

To state it in more direct terms, our requirement for this cut is to simply be able to respond to an inbound **request** (in the sense of a request from a request-response style message).  We'll test this requirement by replacing our native `customerApp` from the last chapter with our `nestMicroservices` app running our new custom transporter.

**Note:** In [part 3](), we'll complete the implementation and have a fully functioning Faye Custom Transporter (server component).  At that point, you'll also have all of the concepts in place to write your own transporter server component, as well as the ability to look inside the built-in transporters (like [the MQTT transporter server]()) and understand what's going on.  That will prepare you for even more adventures, like customizing the Nest built-in transporters to add features&#8212; the subject of my next NestJS microservice tutorial (already underway, and coming very soon)!

### Summary
Here's what it looks like:

![Native App Demo](./assets/native-app-demo.gif 'Native App Demo')
<figcaption><a name="screen-capture-2"></a>Screen Capture 2: Native App Demo</figcaption>

### What's Next

In [Part 3](xxx),

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
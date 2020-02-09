---
published: false
title: "Integrate NestJS with External Services using Microservice Transporters (Part 1)"
description: "tags: nestjs, nest, NATS, microservices, node.js"
series: "NestJS Microservices in Action"
cover_image: "https://dev-to-uploads.s3.amazonaws.com/i/1khd2pysk1o2go9e0oxs.jpg"
canonical_url:
---

*John is a member of the NestJS core team*

### Introduction

This article series covers the topic of integrating NestJS applications with other applications and services.  This can be accomplished in several different ways; the approach covered in this series is to use a combination of [Nest microservices](https://docs.nestjs.com/microservices/basics) and a message broker as the *glue*.  In other words, Nest apps and external apps communicate with each other asynchronously through a message broker.

In these examples, we use [NATS](https://docs.nats.io/) as the message broker. **Note**: some of the concepts here are common to all Nest microservices transporters, while others are specific to NATS.  I try to identify those differences in the text.

### Source Code

Working examples and all source code is available [here](https://github.com/johnbiundo/nest-nats-sample). The articles work best if you follow along with the repository.  Full installation instructions, along with myriad other details, are available [here](https://github.com/johnbiundo/nest-nats-sample/blob/master/README.md).


### Nest Microservices Background

Nest's microservices package is one of those technologies that can be a little hard to wrap your mind around.  Not that it's overly complex &#8212; to the contrary, it presents an application programming model that is quite simple. I think the confusion comes from the use of the term "microservices", which has become so over-used as to be virtually meaningless, and from the fact that Nest's microservices package supports several considerably different use cases:

* Used internally as a **transport layer**, there is little beyond the surface level that a developer needs to understand to effectively use it.  It simply "works", and provides a mechanism for adding features such as scalability and fault tolerance to your Nest application(s).

* Used as an **integration technology**, one needs to dig a bit deeper.  Because the microservices package plays this dual role, when using it for integration there are important implementation details that surface and can get in the way.  Fortunately, as is usually the case, the elegance of the Nest architecture provides mechanisms that make this use case possible (if not immediately obvious). This topic is the focus of this article series.

### Article Series Overview

This article series covers the **integration use case**.  To get there, it first covers some Nest microservices basics.  Some of these basics might be of interest to you even if you are using the package for the first use case described above (inter-Nest-app communication layer).  The articles are organized so you can skip the basics if you want, or if you want to come back later for reference, you can just read the parts you need.

* **Part 1** (this article!) covers introductory material, including the basic microservices communication model, brokers, and how Nest uses publish/subscribe and request/response communication models. It introduces external integration use-cases that will be covered in the series. It also lays out a vocabulary for describing all the moving parts in these sometimes-complex interactions.
* **Part 2** (coming soon) digs a little deeper, extending the vocabulary to describe the modalities in which Nest operates when interacting with external systems. It introduces a pair of simple external (non-Nest, NATS-based) apps that we will use as part of the integration case study.  It introduces the potential challenges arising from Nest's use of a proprietary *message format*.
* **Part 3** (coming soon) describes the Nest approach for solving the message format challenge. It goes on to work through code samples that implement this approach.  It uses the vocabulary introduced earlier to help you keep a simple cognitive model in mind as you address this problem.
* **Part 4** (coming soon) covers some leftovers, some advanced use cases, and some tips and tricks.

I hope you find this article series helpful!

### Part 1: Introduction to Nest Transporters

[NestJS](https://github.com/nestjs/nest) comes with a built-in versatile network communications-oriented subsystem called (perhaps somewhat confusingly) [microservices](https://docs.nestjs.com/microservices/basics). The core of Nest microservices, and the key to understanding them, is what Nest calls **transporters**. Transporters enable you to connect components over a network using a pluggable communications layer and a very simple application-level [message protocol](https://docs.nestjs.com/microservices/basics#request-response). Nest delivers a variety of transporters out-of-the-box, as well as an API allowing developers to build new custom transporters. This combination of architecture and features is extremely powerful.

A concrete example of a built-in Nest transporter is [NATS](https://docs.nats.io/). With the NATS-flavored transporter plugged in, Nest applications can communicate using the NATS messaging system. Since NATS is the intermediary for these communications, Nest can communicate not only with other Nest apps, but also with non-Nest apps, or a combination of both. This leads to four interesting use cases <a name="figure1"></a>(Figure 1 below).

![Transporter Use Cases](./assets/transporter-use-cases.png 'Transporter Use Cases')
<a name="figure1"></a><figcaption>Figure 1: Transporter Use Cases</figcaption>

The current [Nest microservices](https://docs.nestjs.com/microservices/basics) documentation focuses almost entirely on the first use case (Case A). And rightly so. If you simply want to benefit from a robust communication layer (e.g., easily adding load balancing and fault tolerance to increase the horizontal scalability of your Nest application), you really don't need to know about 90% of the material in this article series! (You may still benefit from a deeper understanding of the Nest microservices architecture, however, so I invite you to read on. Much of the material is still relevant background for leveraging Nest microservices).

The rest of this article series focuses mainly on the last three use cases &#8212; namely, using Nest microservices running on NATS to integrate with external systems.

To get there we need to lay some conceptual groundwork. Let's get started.

### Concepts

There's a lot going on in those diagrams, even viewed at the highest level. As soon as we begin diving into the details, the terminology soup can easily get in the way of deeper understanding. So without further ado, let's take a crack at some basic concepts and terminology.

First, let's acknowledge that there are two main flavors of Nest transporters, which I'll refer to as:

- **broker-based**: this includes NATS, as well others like [**Redis**](https://docs.nestjs.com/microservices/redis), [**RabbitMQ**](https://docs.nestjs.com/microservices/rabbitmq), [**MQTT**](https://docs.nestjs.com/microservices/mqtt), and [**Kafka**](https://docs.nestjs.com/microservices/kafka)
- **point-to-point**: this includes [**TCP**](https://docs.nestjs.com/microservices/basics) and [**gRPC**](https://docs.nestjs.com/microservices/grpc)

This article focuses exclusively on the broker-based transporters, so let's discuss that flavor a bit further, using NATS as our case study example. We start with a description of a **broker** in general terms. An installed broker is comprised of several parts:

- the broker **server**: this is an active server process (or possibly replicated servers) that manages publishing and subscribing (and the bookkeeping that goes with it), and delivers messages to clients.
- the **broker client API**: this is delivered in a language-specific package (e.g., a JavaScript flavor, a Java flavor, a Go flavor, etc.) providing an API for accessing the broker from client programs. Client programs include both "native NATS applications" you write with the API (and without benefit of a framework), as well as frameworks like NestJS which use the client API to communicate to the broker.

Message flow diagrams typically only label the broker component (in our case, for example, as a green circle in the middle of various other components), but think of the client API as embedded in the other boxes, connecting them to the broker.

The key advantage of broker-based messaging systems is they allow you to decouple the various application components from each other. Each need only connect to the broker, and can remain unaware of the existence of, location of, or implementation details of other components. The only thing the components need to share is a message protocol.

Nest makes use of a small set of features that are typically available across most brokers. This is a bit of an over-generalization &#8212; one we'll revisit later on &#8212; but for now, let's examine the common broker features that Nest relies on.

### Broker Message Protocol

From Nest's perspective, brokers provide a basic message-oriented communication protocol that is usually described as **publish/subscribe**. Publish/subscribe can be generally understood in terms of the following diagram, with the red circles indicating the order of events in a publish/subscribe "conversation".

![Broker Message Protocol](./assets/broker-message-protocol.png 'Broker Message Protocol')
<figcaption><a name="figure2"></a>Figure 2: Broker Message Protocol</figcaption>

In that diagram, each client component is either a _publisher_ or a _subscriber_. The salient point is that subscribers "register interest in a **topic**" and publishers publish messages about a topic. The broker sits in the middle and performs the following functions:

- keeps track of subscribers (by managing a list of **subscriptions**)
- receives published messages
- forwards published messages to all interested subscribers (based on matching topics between published messages and subscriptions)

One thing obviously missing from this diagram is the [request/response](https://en.wikipedia.org/wiki/Request%E2%80%93response) style communication model. This communication style is useful when we want to verify receipt of a published message and/or receive a response from whoever consumes the message. In fact, this is the style of message implicit in [Figure 1](#figure1) above (where we can infer that `'get-customers'` is a **request** and the array of customers is a **response**). How do we get from **publish/subscribe** to **request/response**?

Nest (as well as some of the brokers natively, including NATS as we'll soon see) solve this problem in a handy way, by building a request/response capability **on top of the publish/subscribe model**. Let's say component A wishes to "get customers" from component B, which has access to a customer DB. Component A can publish a "get customers" message, and (assuming it has subscribed to that topic) component B receives it, queries the customer DB for a list of customers, and sends a response message. The response message is where the magic happens. In order for B to respond to A, they both must do a few things, agreed upon by convention:

- A chooses a **response topic** (sometimes called a **reply subject**)
- A subscribes to the response topic
- A passes the response topic as part of the initial message
- B uses the response topic as the topic of its own subsequent response message

In other words, if a response topic is included in a message received by a subscriber, the subscriber publishes a response message on that response topic. Thus, in our example, B publishes its response, including a payload containing the customer list, on the response topic it received in the "request" message. All very tidy!

### A Vocabulary for Transporter Use Cases

With this in mind, we can give names to the two different types of messages our system handles, and also describe the **roles** of the participants in those messages. Don't worry, we're almost done with the preliminaries! These terms will all become second nature pretty quickly, but it turns out to be extremely useful to have a vocabulary that will let us quickly navigate a bunch of similar-looking data flow diagrams, and keep track of where our custom code fits in!

The first kind of message we can exchange based on the above capabilities is called an **event**. Any given component participating in event-based messaging can be either:

- an event **emitter** - meaning it _publishes_ a message with a topic (and an optional payload). An event emitter is a pure message publisher.
- an event **subscriber** - meaning it registers interest in a topic, and receives messages (forwarded by the broker) when a message matching that topic is published by an emitter.

The second kind of message we can exchange is called a **request/response** message. In this exchange, a participating component can be either:

- a **requestor** - meaning it _publishes_ a message it intends to be treated as a request, and it also takes the conventional steps described above &#8212; namely, _subscribing_ to a response topic and including that response topic in the message it publishes.
- a **responder** - meaning it _subscribes_ to a topic it intends to treat as an incoming request, it produces a result, and it _publishes_ a response, including the result payload, to the response topic it received on the inbound request.

So requestors are a special case of publishers, and we use this term to remember that they **also** await a response. And responders are a special case of subscribers, and we use this term to remember that they are **also** publishers of a response. Make sense? If not, don't worry, it will all come together soon.

### Integration Use Cases

Most of the above is background that "just works" in an all-Nest world.  Let's turn our attention now to integrating Nest apps with non-Nest apps using NATS as the communication intermediary.

In <a href="#figure1">Figure 1</a>, cases B, C, and D illustrate the different topologies for Nest-based apps to integrate with non-Nest apps using NATS. Note, we'll use the generic term **"app"** to describe these communicating components. Let's briefly motivate these use cases.

In case B, we could be migrating a legacy app service, responsible for managing our customer DB, to Nest. In this case, Nest needs to play the role of **responder** that we described above. Going forward, we call this the **Nest as responder** use case.

In case C, we could be writing a new app, in Nest, that needs to query our customer DB using the existing (legacy) service. In this case, Nest needs to play the role of **requestor** that we described above. Going forward, we call this the **Nest as requestor** use case.

In case D, we could be somewhere in the middle of a complex migration process, where both types of communications are happening. Nest apps need to be able to speak to both other Nest apps and non-Nest apps, all via NATS. Non-nest apps similarly need to be able to talk to both types of apps. Going forward, we call this the **Nest as duplex requestor/responder** use case.

We'll examine each use case in more detail, including sample code you need to make this work (reminder [full repository with usage notes here](https://github.com/johnbiundo/nest-nats-sample)). Before we can do that, we need to understand a little more about how Nest components interact with the broker. We cover this in the next installment in the series.

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
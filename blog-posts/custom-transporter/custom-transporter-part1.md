---
published: False
title: "Part 1: Introduction and Setup"
description: "tags: nestjs, nest, faye, microservices, node.js"
series: "Advanced NestJS Microservices"
cover_image:
canonical_url:
---

*John is a member of the NestJS core team*

xxx longdash:
&#8212;

### Introduction

This article series covers the topic of building a custom **transporter** for the NestJS microservices subsystem. If you haven't already read it, please check out my [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series, where I cover many of the basics of the NestJS microservices subsystem architecture, and establish the terminology used all my Nest Microservices articles.  I'll assume a good understanding of those basics in this article series.

Let's start with the *Why?* question.  The Nest microservices package provides a communications layer abstraction that makes it easy for applications (both Nest and non-Nest) to communicate over a wide variety of what are called **transporters**.  Nest comes with a variety of built-in transporters, including NATS, RabbitMQ, Kafka, and others. But what if you want to build your own transporter &#8212; say for ZeroMQ, Google Cloud Pub/Sub, Kinesis or ActiveMQ? If that's your desire, you've landed on the right article!  Or, if you just want to understand more about how the "magic" works, read on.

A related question is *How can I extend the capabilities of existing Nest transporters?*  For example, maybe you'd like to use the **QoS** feature with the [MQTT]() transporter.  That's a topic I'll cover in another upcoming series (already under development!), but this series provides a baseline for addressing that requirement.

### Article Series Overview

Part 1
Part 2
...

Let's get started!

### Source Code

Working examples and all source code is available [here](https://github.com/johnbiundo/xxx). The articles work best if you follow along with the repository.  Full installation instructions, along with many other details, are available [here](https://github.com/johnbiundo/#xxx).

### Introduction to Faye

For this case study, we'll build our custom transporter to work with the [Faye](https://faye.jcoglan.com/) message broker.  This is a simple OSS JavaScript publish/subscribe message broker that runs nicely on Node.js.  Getting it up and running, and understanding its API, are simple, yet it provides [all the features Nest needs](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3#broker-message-protocol) to build a transporter.

Faye has a very simple &#8212; virtually canonical &#8212; publish/subscribe protocol, as depicted in Figure 1 below.


![Faye Message Protocol](./assets/faye-protocol.png 'Faye Message Protocol')
<figcaption><a name="figure1"></a>Figure 1: Faye Message Protocol</figcaption>

As we've done in the [NestJS Microservices in Action](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) series, we'll start by building simple native requestor and responder apps.  To run the code in this section, [read these instructions]() for the github repository.

As covered in the previous article, one of the challenges Nest microservices deals with is to layer a request/response message style **on top of** publish/subscribe semantics.  In other words, Nest requestors need to be able to run code like:

```ts
xxx
async function addCustomer(name) {
  const payload = getPayload('/add-customer', { name });
  try {
    await client.publish('/add-customer', payload);
    console.log(
      `<== Publishing add-customer event with payload:\n${JSON.stringify(
        payload,
      )}\n`,
    );
  } catch (error) {
    console.log('Error publishing event: ', error);
  }
}
```

Since Faye (as well as most brokers) only supports publish/subscribe, we need to follow a simple recipe.  Here's a recap of the *generic pattern* for how this is often done.

Let's say component A wishes to "get customers" from component B, which has access to a customer DB. Component A can publish a "get customers" message, and (assuming it has subscribed to that topic) component B receives it, queries the customer DB for a list of customers, and sends a response message. The response message is where the magic happens. In order for B to respond to A, they both must do a few things, agreed upon by convention:

- A chooses a **response topic** (sometimes called a **reply subject**)
- A subscribes to the response topic
- A passes the response topic as part of the initial message
- B uses the response topic as the topic of its own subsequent response message

Nest makes this a bit easier by automatically choosing the response topic name for you. In fact, Nest builds **two** topics from each **pattern** you declare.  For example, if you define a Nest microservice responder message pattern like this:

```ts
@MessagePattern('get-customers')
```

Nest uses the term **channel** as a generic name for topics (also called subjects), and to help disambiguate them from the "user land" concept of a **pattern**.  So, internally, Nest builds two channels from the above pattern:

* `'get-customers_ack'` - this is the physical topic name we'll use in the Faye transporter to publish/subscribe to **requests** messages
* `'get-customers_res'` - this is the physical topic name we'll use in the Faye transporter to publish/subscribe to **response** messages


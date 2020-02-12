---
published: true
title: "Part 3: Techniques for Application Integration with NestJS and NATS"
description: "tags: nestjs, nest, NATS, microservices, node.js"
series: "NestJS Microservices in Action"
cover_image: "https://dev-to-uploads.s3.amazonaws.com/i/7pdohpezsqthfk0ieab8.png"
canonical_url:
---

*John is a member of the NestJS core team*

### Part 3: Solving the Message Incompatibility Problem

In [Part 2](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-2-3hgd) of this series, we described the challenges of integrating Nest apps with non-Nest apps using NATS as an intermediary.

In this article, we explore solutions to that problem.

### Get the Code

Reminder: all the code for these articles is available [here](https://github.com/johnbiundo/nest-nats-sample), with complete instructions [here](https://github.com/johnbiundo/nest-nats-sample#sample-repository-for-nestnatsmicroservice-article-series).

### Serialization and Deserialization

Serialization and deserialization are the steps Nest uses to translate between message formats. In other words, Nest receives inbound messages (from the broker), **deserializes** them (converts arbitrary message formats into Nest message formats), and then processes them. For outbound messages, the process is reversed: the last step before sending a message to the broker is to (optionally) **serialize** it (convert from Nest message format to some other format).

The nifty part is that while Nest does all this automatically, it also provides hooks where you can easily plug in your own serializers and deserializers to handle arbitrary message formats. Serializers and deserializers are simply classes that implement a defined interface.

Since requestors and responders perform their own unique roles (each with their own context) and each are both message senders and message receivers, **there are four hooks where serialization/deserialization take place**. These are examined in the recipes below. If there's one section of this article series that you should bookmark for future reference, this is it. Each recipe describes exactly where you should look to handle a serialization/deserialization task, including the full context you need to understand why and how to do it.

#### Serialization/Deserialization Hooks and Recipes

Let's examine the two different contexts (requestors and responders) in which we need to serialize/deserialize. For each, we'll examine handling inbound and outbound messages.

##### Nest as Requestor

Nest requestors live in the context of a `ClientProxy` instance. It follows that we customize serialization/deserialization behavior by modifying the `ClientProxy` behavior. For example:

```typescript
// app.controller.ts
@Client({
  transport: Transport.NATS,
  options: {
    url: 'nats://localhost:4222',
    serializer: new OutboundRequestSerializer(),
    deserializer: new InboundResponseDeserializer(),
  }
})
client: ClientProxy;
```

In the code fragment above, `OutboundRequestSerializer` is the class that answers the question *"how do does my Nest requestor format outgoing requests so that an external app can process them?"*, while `InboundResponseDeserializer` is the class that addresses *"how does my requestor translate incoming external responses so that Nest can process them?"*.

**Note**: you can choose any name for these classes, but trust me, it's **highly recommended** that you choose a naming convention similar to the ones above to keep things straight!

##### Nest as Responder

Nest responders live in the context of a microservice instance. It follows that we customize serialization/deserialization behavior by modifying the microservice instance behavior. For example:

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.NATS,
    options: {
      url: 'nats://localhost:4222',
      deserializer: new InboundMessageDeserializer(),
      serializer: new OutboundResponseSerializer(),
    },
  });
  app.listen(() => console.log('Microservice is listening...'));
}
```

So `InboundMessageDeserializer` is the class that answers the question *"how does my responder translate incoming external messages so that Nest can process them?"*, while `OutboundResponseSerializer` is the class that handles *"how does my responder format responses so that the external app can understand them?"*.

This feature results in the following diagrams &#8212; ones that remove all those ugly red X's in the <a href="https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-2-3hgd#figure4">last diagram</a>!

Here's the diagram to keep in mind when writing **serializers/deserializers for Nest responders**.

![Nest Responder (De)serialization](./assets/responder-serialization.png 'Nest Responder (De)serialization')
<a name="figure1"></a><figcaption>Figure 1: Nest Responder (De)serialization</figcaption>

In the above diagram, the Nest responder serializer and deserializer are configured in the `NestFactory.creatMicroservice()` call.

And here's the corresponding diagram for **Nest requestors**.

![Nest Requestor (De)serialization](./assets/requestor-serialization.png 'Nest Requestor (De)serialization')
<a name="figure2"></a><figcaption>Figure 2: Nest Requestor (De)serialization</figcaption>

In the above diagram, the Nest requestor serializer and deserializer are configured in the `ClientProxy` configuration (decorator or injectable, depending on [which method](https://docs.nestjs.com/microservices/basics#client) you use).

#### Implementing Serializers and Deserializers

Before we start writing serializer/deserializer classes, let's take a look at their interfaces.

##### Serializer Interface

A serializer class needs to implement the `serialize()` method, which takes a single argument &#8212; the outbound payload to be serialized &#8212; and returns the serialized payload.

```typescript
export interface Serializer<TInput = any, TOutput = any> {
  serialize(value: TInput): TOutput;
}
```

We'll now implement an **identity serializer** for our Nest responder as an exercise. An identity serializer simply returns the same message it receives. In addition to exercising the interface, we can use it to "spy on" the otherwise silent serialization function that Nest always performs on our behalf.

##### Responder Identity Serializer Implementation

Here, we'll be working in the *nestMicroservice* project (i.e., nest-nats-sample/nestMicroservice directory). Take a look at the `src/common/serializers/outbound-response-identity.serializer.ts` file:

```typescript
// src/common/serializers/outbount-response-identity.serializer.ts
import { Serializer, OutgoingResponse } from '@nestjs/microservices';
import { Logger } from '@nestjs/common';

export class OutboundResponseIdentitySerializer implements Serializer {
  private readonly logger = new Logger('OutboundResponseIdentitySerializer');
  serialize(value: any): OutgoingResponse {
    this.logger.debug(
      `-->> Serializing outbound response: \n${JSON.stringify(value)}`
    );
    return value;
  }
}
```

Now simply plug this serializer into the `main.ts` file for the Nest responder app.  If you're following along with the github repository, the `main.ts` file will have all of this code (and more that we'll get to soon) in place, but commented out.  Simply uncomment the line shown below.

```typescript
// src/main.ts
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.NATS,
  options: {
    queue: 'customers',
    url: 'nats://localhost:4222',
    /**
      * Use the "Identity" (de)serializers for observing messages for
      * nest-only deployment.
      */
    serializer: new OutboundResponseIdentitySerializer(),
  },
});
```

If you now send some requests to the *nestMicroservice* app from the *nestHttpApp* ([here's how](https://github.com/johnbiundo/nest-nats-sample#running-the-all-nest-configuration)), you'll see logging information that looks something like this, showing you the internal layout of the outbound message:

```bash
[Nest] 8786   - 02/04/2020, 10:19:04 AM   [OutboundResponseIdentitySerializer] -->> Serializing outbound response:
{"err":null,"response":[],"isDisposed":true,"id":"41da72e8-09d0-4720-9404-bd0977b034a0"}
```

##### Deserializer Interface

Now let's take a look at the required interface for a deserializer.

```typescript
export interface Deserializer<TInput = any, TOutput = any> {
  deserialize(value: TInput, options?: Record<string, any>): TOutput;
}
```

A deserializer class needs to implement the `deserialize()` method, which takes a two arguments &#8212; the payload to be deserialized and an optional `options` object &#8212; and returns the deserialized payload.

The `options` object contains metadata about the incoming message. For NATS, the object contains the value of the `replyTo` message property, if one exists.

Let's implement a deserializer. To start with, we'll build an "identity deserializer" that just logs out the contents of any incoming message without doing any transformation on it.

##### Responder Identity Deserializer Implementation

We're still working on the Nest responder &#8212; the *nestMicroservice* project (i.e., nest-nats-sample/nestMicroservice directory). Take a look at the `src/common/deserializers/inbound-message-identity.deserializer.ts` file:

```typescript
// src/common/serializers/inbound-message-identity.deserializer.ts
import { ConsumerDeserializer, IncomingRequest } from '@nestjs/microservices';
import { Logger } from '@nestjs/common';

export class InboundMessageIdentityDeserializer
  implements ConsumerDeserializer {
  private readonly logger = new Logger('InboundMessageIdentityDeserializer');

  deserialize(value: any, options?: Record<string, any>): IncomingRequest {
    this.logger.verbose(
      `<<-- deserializing inbound message:\n${JSON.stringify(
        value
      )}\n\twith options: ${JSON.stringify(options)}`
    );
    return value;
  }
}
```

Like our identity serializer, we can quickly plug in an identity deserializer to spy on the incoming messages from our requestor.  Again, if you're following along using the github repository, simply comment out the lines so the active (de)serializers match those shown below.

```typescript
// src/main.ts
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.NATS,
  options: {
    queue: 'customers',
    url: 'nats://localhost:4222',
    /**
      * Use the "Identity" (de)serializers for observing messages for
      * nest-only deployment.
      */
    serializer: new OutboundResponseIdentitySerializer(),
    deserializer: new InboundMessageIdentityDeserializer(),
  },
});
```

If you again send some requests to the *nestMicroservice* app from the *nestHttpApp* ([like this](https://github.com/johnbiundo/nest-nats-sample#running-the-all-nest-configuration)), you'll see logging information that looks something like this, showing you the layout of the inbound message:

```bash
[Nest] 8786   - 02/04/2020, 10:19:04 AM   [InboundMessageIdentityDeserializer] <<-- deserializing inbound message:
{"pattern":"get-customers","data":{},"id":"41da72e8-09d0-4720-9404-bd0977b034a0"}
        with options: {"channel":"get-customers","replyTo":"_INBOX.IDML09N4JB6W0MF4P6GBPB.IDML09N4JB6W0MF4P6GCX2"}
```

Notice the value for the `replyTo` field in the logging output. You can probably guess what that is. This is a detail that's going to come in handy down the line, so file that away for future reference.

### Implementing the Integration Use Cases

We've nearly arrived at our final destination. With the understanding we've gained, we can specify what we need to complete our integration use cases from [Figure 1 in Part 1](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3).

Here are the requirements:
1. The *nestHttpApp*, in its role as a requestor, must implement:
   * A) <a name="1a"></a>an **outbound message external serializer** that translates a Nest formatted request into a request understood by our external service. For example:
      * From Nest format
          * topic: `'get-customers'`
          * reply topic: `'_INBOX.XVRL...'`
          * payload: `{pattern: 'get-customers', data: {}, id: 'abc...'}`
      * To external format
          * topic: `'get-customers'`
          * reply topic: `'_INBOX.XVRL...'`
          * payload: `{}`
   * B) <a name="1b"></a>an **inbound response external deserializer** that translates an external response into a format understood by Nest. For example:
      * From external format
          * topic: `'_INBOX.XVRL...'`
          * payload: `{customers: [{id: 1, name: 'nestjs.com'}]}`
      * To Nest format
          * topic:`'_INBOX.XVRL...'`
          * payload: `{err: undefined, response: {customers: [{id: 1, name: 'nestjs.com'}]}, isDisposed: true}`

2. The *nestMicroservice* app, in its role as a responder, must implement:
    * A) <a name="2a"></a>an **inbound message external deserializer** that translates an external request into a format understood by Nest.  For example:
      * From external format
          * topic: `'get-customers'`
          * reply: `'_INBOX.XVRL...'`
          * payload: `{}`
      * To Nest format
          * topic `'get-customers'`
          * reply topic: `'_INBOX.XVRL...'`
          * payload: `{pattern: 'get-customers', data: {}, id: 'abc...'}`
    * B) <a name="2b"></a>an **outbound response external serializer** that translates a Nest formatted response into a response understood by our external service.  For example:
      * From Nest format
          * topic: `'_INBOX.XVRL...'`
          * payload: `{err: undefined, response: {customers: [{id: 1, name: 'nestjs.com'}]}, isDisposed: true}`
      * To external format
          * topic: `'_INBOX.XVRL...'`
          * payload: `{customers: [{id: 1, name: 'nestjs.com'}]}`

With that in mind, let's get busy!  The **bolded** descriptions above inform the names of our classes.

For [requirement 1-A](#1a), we have `src/common/serializers/outbound-message-external.serializer.ts` in our *nestHttpApp* project.  Here's the code.  The comments explain its intent.

```typescript
// src/common/serializers/outbound-message-external.serializer.ts
import { Logger } from '@nestjs/common';
import { Serializer } from '@nestjs/microservices';

export class OutboundMessageExternalSerializer implements Serializer {
  private readonly logger = new Logger('OutboundMessageExternalSerializer');
  serialize(value: any) {
    this.logger.debug(
      `-->> Serializing outbound message: \n${JSON.stringify(value)}`,
    );

    /**
     * Here, we are merely "unpacking" the request payload from the Nest
     * message structure and returning it as a "plain" top-level object.
     */
    return value.data;
  }
}
```

For [requirement 1-B](#1b), we have `src/common/deserializers/inbound-response-external.deserializer.ts` in our *nestHttpApp* project.  Here's the code.  The comments explain its intent.

```typescript
// src/common/deserializers/inbound-response-external.deserializer.ts
import { WritePacket, Deserializer } from '@nestjs/microservices';
import { Logger } from '@nestjs/common';

export class InboundResponseExternalDeserializer implements Deserializer {
  private readonly logger = new Logger('InboundResponseExternalDeserializer');

  deserialize(value: any): WritePacket {
    this.logger.verbose(
      `<<-- deserializing inbound response:\n${JSON.stringify(value)}`,
    );

    /**
     * Here, we wrap the external payload received in a standard Nest
     * response message.  Note that we have omitted the `id` field, as it
     * does not have any meaning from an external responder.  Because of this,
     * we have to also:
     *   1) implement the `Deserializer` interface instead of the
     *      `ProducerDeserializer` interface used in the identity deserializer
     *   2) return an object with the `WritePacket` interface, rather than
     *      the`IncomingResponse` interface used in the identity deserializer.
     */
    return {
      err: undefined,
      response: value,
      isDisposed: true,
    };
  }
}
```

For [requirement 2-A](#2a), we have `src/common/deserializers/inbound-message-external.deserializer.ts` in our *nestMicroservice* project.  Here's the code.  The comments explain its intent.

```typescript
// src/common/serializers/inbound-message-external.deserializer.ts
import { Logger } from '@nestjs/common';
import * as uuid from 'uuid/v4';
import { ConsumerDeserializer } from '@nestjs/microservices';

export class InboundMessageExternalDeserializer
  implements ConsumerDeserializer {
  private readonly logger = new Logger('InboundMessageExternalDeserializer');
  deserialize(value: any, options?: Record<string, any>) {
    this.logger.verbose(
      `<<-- deserializing inbound external message:\n${JSON.stringify(
        value,
      )}\n\twith options: ${JSON.stringify(options)}`,
    );

    /**
     * Here, we merely wrap our inbound message payload in the standard Nest
     * message structure.
     */
    return {
      pattern: undefined,
      data: value,
      id: uuid(),
    };
  }
}
```

For [requirement 2-B](#2b), we have `src/common/serializers/outbound-response-external.serializer.ts` in our *nestMicroservice* project.  Here's the code.  The comments explain its intent.

```typescript
// src/common/serializers/outbound-response-external.serializer.ts
import { Serializer, OutgoingResponse } from '@nestjs/microservices';
import { Logger } from '@nestjs/common';

export class OutboundResponseExternalSerializer implements Serializer {
  private readonly logger = new Logger('OutboundResponseExternalSerializer');
  serialize(value: any): OutgoingResponse {
    this.logger.debug(
      `-->> Serializing outbound response: \n${JSON.stringify(value)}`,
    );

    /**
     * Here, we are merely "unpacking" the response payload from the Nest
     * message structure, and returning it as a "plain" top-level object.
     */

    return value.response;
  }
}
```

#### Enabling the External (De)serializers

The last step is to plug in these (de)serializers in at the appropriate "hook points".  We've seen this before.  For the `nestHttpApp`, this is done wherever you configure the `ClientProxy`.  In our case, for ease of code organization, we've done this with the `@Client()` decorator in the `src/app.controller.ts` file.  To enable the external (de)serializers, update that file to look like this:

```typescript
// src/app.controller.ts
  @Client({
    transport: Transport.NATS,
    options: {
      url: 'nats://localhost:4222',
      /**
       * Use the "Identity" (de)serializers for observing messages for
       * nest-only deployment.
       */
      // serializer: new OutboundMessageIdentitySerializer(),
      // deserializer: new InboundResponseIdentityDeserializer(),

      /**
       * Use the "External" (de)serializers for transforming messages to/from
       * (only) an external responder
       */
      serializer: new OutboundMessageExternalSerializer(),
      deserializer: new InboundResponseExternalDeserializer(),
    },
  })
  custClient: ClientProxy;
```

For the `nestMicroservice`, this is done in the `src/main.ts` file.  To enable the external (de)serializers, update that file to look like this:

```typescript
// src/main.ts
  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.NATS,
    options: {
      queue: 'customers',
      url: 'nats://localhost:4222',
      /**
       * Use the "Identity" (de)serializers for observing messages for
       * nest-only deployment.
       */
      // serializer: new OutboundResponseIdentitySerializer(),
      // deserializer: new InboundMessageIdentityDeserializer(),

      /**
       * Use the "External" (de)serializers for transforming messages to/from
       * an external requestor
       */
      serializer: new OutboundResponseExternalSerializer(),
      deserializer: new InboundMessageExternalDeserializer(),
    },
  });
```

At this point, you should be able to send and receive requests between any combination of apps.  For example, from *nestHttpApp* to *customerService*, from *customerApp* to *nestMicroservice*, and all other combinations.  I highly recommend you do so now, using the instructions from [here](https://github.com/johnbiundo/nest-nats-sample#running-the-all-nest-configuration) and [here](https://github.com/johnbiundo/nest-nats-sample#running-the-all-native-app-configuration) to run all components at the same time\*. Pay close attention to the log outputs which show how (de)serialization is happening at each step.

\*<a name="queues"></a>We haven't described the use of `queues`, but when you start running the full mixed configuration (Nest and non-Nest components), one thing you'll notice is that requests will be randomly routed between the *nestMicroservice* app and the *customerService* app.  This is because they are members of a **NATS queue**. If you want to send a request to a particular responder, you'll need to shut down the other one to guarantee the message is sent there.  I'll discuss NATS queues in more detail the final article of this series.

Here's a nice surprise! We've magically handled [Figure 1 Case D](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) along the way.  If you think carefully about it, this is both not a surprise, and has a somewhat strict limitation.  The reason this works is that we're now serializing and deserializing to and from a canonical external format.  So **all** requests look like they come from an external requestor, and **all** responses look like they come from an external responder.  With this in place, our serializers and deserializers work in all combinations.  We've created a "lingua franca" message protocol.  The limitation with this approach is perhaps a subtle one, and is something we'll address later (see bullet point #2 in **What's Next?** below).

### Conclusion :rocket:

We've covered a lot of ground.  Hopefully you've got both a conceptual framework for how to think about integrating external applications with Nest apps using tools like Nest microservices and the NATS message broker, and some boilerplate code you can work with.  Some of these concepts also translate directly to other Nest microservice transporters, but as you can imagine, the details are different.  Holler in the comments if you'd like me to cover other transporters in this fashion in another series, and I'll see if I can do so.

### What's next? :question:

There are a few remaining subtle topics to cover, which I'll do in the next installment of this series. As a teaser, these include:

1. Can we run both the *nestMicroservice* app and the external *customerService* app at the same time, and load balance requests across them?  Spoiler: yes!  In fact, we already did so, as mentioned [above](#queues). This is simple to do with NATS [distributed queues](https://docs.nats.io/nats-concepts/queue), which we'll cover briefly in the next article.
2. We covered the case where we're running our modified *nestHttpApp* in this [hybrid](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3) (Nest and non-Nest) environment.  What happens if we have a mixture of Nest apps that are unmodified  &#8212;  that is, they are deployed apps that we don't want to touch to implement our "lingua franca" message protocol (i.e., they're running out-of-the-box standard de(serializers))?  Can they be made to play nicely in this hybrid environment?  Spoiler: yes!
3. What about **events**?  We've covered what seems to be the trickier case here with request/response style messaging, but how about plain old events?  In fact, you may have noticed we've built a feature for **adding a customer** (see the `'add-customer'` messages and handlers sprinkled through our code).  Go ahead and try them now. You'll see you have mixed results. Can we make events cooperate in this hybrid environment?  Spoiler: yes!

Stay tuned for the answers to these and other exciting questions in our next installment! :smiley:

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.

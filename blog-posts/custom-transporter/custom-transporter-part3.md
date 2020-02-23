---
published: False
title: "Part 3: Completing the Server Component"
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

All of the code in these articles is available [here]().  As always, these tutorials work best if you follow along with the code.  The [READM]() covers all the steps you need to get the repository, build the apps, and follow along.  It's easy!  I strongly encourage you to do so.

#### Building the Apps for this Article

### Overview

I ended the last article with a discussion of the shortcomings of our "Take 1" (first iteration) implementation of the Faye Custom Transporter.  In this article, we'll address those shortcomings. Most of this work is fairly straightforward, but I want to take a moment to discuss the "Observable issue" a bit further.

You **could** actually ignore this issue entirely.  The server component we built in the last chapter is functional, and we'll clean up a few loose ends here to make it typesafe, etc.  However, by ignoring proper handling of Observables, you'd be leaving out a feature that you may not miss until you **really** need it.  Plus, let me hit you with the full commercial:
- It's super fun to understand how it works, and if you're not an `RxJS` whiz, that won't stop you **and** you may learn a few nifty tricks about `RxJS` along the way
- It builds a huge appreciation and understanding of the elegance of the Nest framework
- It's really, really easy to implement, once you get on the right path
- The feature provides some amazing benefits, virtually for free
- Your transporter really **should** be plug-and-play compatible with other transporters to be a good Nest citizen

I could go on and on! And in fact, I do sort of go on, but I've decided to encapsulate all the [concrete examples and details here]() to try to keep this tutorial on track.  Suffice to say, I strongly encourage you to read that section to both understand why this is so useful, and to possibly get inspired to use Nest microservices in some new and interesting ways.

With all that said, let's jump in!

### Addressing Observables



### What's Next

In [Part 4](xxx), we cover:
* A
* B
* C

Feel free to ask questions, make comments or suggestions, or just say hello in the comments below. And join us at [Discord](https://discord.gg/nestjs) for more happy discussions about NestJS. I post there as _Y Prospect_.
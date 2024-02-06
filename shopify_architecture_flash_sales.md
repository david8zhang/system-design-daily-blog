---
author: David Zhang
date: 02/05/2024
---

# How Shopify Handles The World's Biggest Flash Sales Without Exploding

## Background Info

[Original QCon tech talk video](https://www.youtube.com/watch?v=MV5Kdwzwcag)

Flash sales on Shopify aren't like those Black Friday stampedes you see IRL. Those are seasonal, whereas on Shopify, these shopping frenzies can happen at any time. For example, online brands will generate tons of hype for limited edition product drops and subsequently bombard Shopify servers with traffic when the product becomes available.

This means Shopify's architecture has to be robust and scalable, especially as the number of merchants grows.

## Shopify's tech stack

- MySQL, Redis, Memcached for storing data
- Mainly Ruby on Rails for backend, tho some Go, Lua for performance critical components
- React and React Native with GraphQL APIs on the frontend

Some other things to note:

- Main rails backend is a monolith that gets deployed 40 times per day
- Large percentage of the workforce is fully remote and globally distributed

## Shopify Data Architecture

Shopify is organized into _pods_. A pod contains data for one or more shops and is a "complete version of Shopify that can run anywhere in the world". Think of it like an enhanced data shard.

The pods themselves are stateful and isolated from one another. In contrast, the web workers and jobs that interact with them are stateless and can be shared. This means more web workers can be allocated to a particular pod during large traffic spikes (e.g. in the event of a flash sale).

Pods are also replicated across regions, with active pods in one region being inactive in another. This enables failovers in the event of catastrophic outages.

![Shopify pod architecture](https://firebasestorage.googleapis.com/v0/b/system-design-daily.appspot.com/o/shopify-pod-architecture.png?alt=media&token=101229b6-5aaa-42e9-9c11-1b8a1669eb60)

### The lifecycle of a request

1. A request comes in to some merchant's online store, e.g. `tshirthero.shopify.com`
2. Request goes to [OpenResty](https://openresty.org/en/), an open source nginx distribution with Lua scripting support.
   - Used for [load balancing](/topic/12_load_balancing) and bot/scalper detection.
3. Request gets routed to the appropriate region where the pod corresponding to the shop is active
4. Rails application running in that region receives the request and handles it

## An Aside on Why Credit Card Payments Are A Bitch

"PCI-DSS", short for the Payment Card Industry Data Security Standard, sets a bunch of guidelines for securely storing and handling credit card information.

This presents a few problems for Shopify

1. Rails app is a monolith being deployed 40 times a day. Auditing every time isn't feasible.
2. Shops can be customized via HTML and Javascript, and Shopify Apps can add additional functionality to stores. Don't wanna expose any credit card info to that stuff.

The solution? Isolate redit card payment data from the frontend and the Rails monolith entirely.

When a user makes a purchase, the following occurs:

1. Credit card info is handled by a web form hosted in an iFrame, isolating it from the rest of the (potentially merchant-controlled) Javascript / HTML.

2. The information is sent over to a service called CardSink, which encrypts and stores it, responding with a Card token back to the client

3. The web client sends that Card token and order info to Shopify backend monolith

4. The backend monolith calls another service called CardServer with the token and metadata.

5. CardServer uses that token and metadata to decrypt the credit card info and uses it to send a request to the appropriate payment processor.

6. Payment processor handles payment authorization, returns a success/declined response to the Shopify monolith, which converts it into an order.

Here's a nice diagram visualizing the request flow:

![shopify-credit-card-processing](https://firebasestorage.googleapis.com/v0/b/system-design-daily.appspot.com/o/credit-card-processing.png?alt=media&token=1401317e-679d-4423-a91d-602c59297a90)

### Idempotency

Furthermore, credit card transactions need to be **idempotent**, meaning multiple duplicate requests being sent and processed should yield the same result as sending and processing single request. This is for obvious reasons - if we double charge a credit card we get a pissed off customer.

Shopify does this by including an _idempotency key_, which is a unique key that the server uses to recognize subsequent retries of the same request. They also developed a library for creating idempotent actions, enabling devs to describe how to store state and retry requests.

## Scaling Pods

Application code only operates on shops. It doesn't care what pods these shops correspond to, meaning pods can be horizontally scaled to handle more load. However, as individual shops grow, pods need to be rebalanced to better utilize system resources.

A shop is moved from one pod to another via the following process:

1. The MySQL rows corresponding to that shop's _existing_ data is just copied over to another MySQL living in another pod.
2. New incoming data is replicated via a "bin log", which is a stream of events for every row that is modified.
   - Shopify open sourced a tool for doing this called [Ghostferry](https://github.com/Shopify/ghostferry), which is written in Go
3. Once everything is in sync, the shop is [write-locked](https://systemdesigndaily.com/topic/03_ACID-transactions?subtopic=05_two_phase_locking), with incoming writes being queued and Redis jobs being migrated.
4. Once that's done, the route table gets updated and the lock is removed.
5. The shop data on the old pod is deleted asynchronously.

This whole process is super fast, with less than 10 seconds of downtime on average, with less than 20 seconds for larger stores.

## Scaling The Storefront Rendering Layer

The rendering logic was abstracted out of the old rails monolith and was rewritten as a new rails application completely from scratch. This enabled it to scale independently from the other parts of Shopify. OpenResty routing and Lua scripting ensured this process was completed with no downtime.

Over the past few years, there's also been a rise of "headless" commerce, reducing the rendering load for Shopify. Tech-savvy merchants are starting to roll out their own frontends as single page React applications.

## Load Testing

Load-testing is done with an internal tool called Genghis, which spins up a bunch of worker VMs that execute Lua scripts. Lua scripts can describe end to end user behavior, like browsing, adding to cart, end checking out.

Genghis is run weekly against benchmark stores saved in every pod. It's also run against a CardServer which forwards these requests to a benchmark gateway (written in Go), to prevent spamming payment processors. The benchmark gateway responds with both successful and failed payments, with a distribution of latencies mimicking that of real production traffic.

## Resiliency

Shopify also created a **resiliency matrix** which describes the dependencies of their system and possible failures that can occur. They use this to plan gamedays, where they simulate outages to check if their alerting mechanisms fire and if the right SOPs are in place.

Shopify also uses [circuit breakers](/topic/13_software_architecture?subtopic=04_microservices_and_fault_tolerance) to improve service uptime and prevent cascading failures. They use an internally developed library called Semian to implement these in Ruby's HTTP client.

In addition, Shopify developed a tool called [Toxiproxy](https://github.com/Shopify/toxiproxy), which enables service failures and latency to be incorporated into unit tests. Toxiproxy consists of a TCP proxy written in Go and a client communicating with the proxy over HTTP. Developers can then simulate latency and failures by routing all connections through Toxiproxy.

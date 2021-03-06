---
layout: default
---

# Event driven architecture

<em class="sub-heading">-</em>

By Per Ökvist ([@per_okvist](https://twitter.com/per_okvist/))<br/>
Please give feedback and report issues on the [GitHub repository](https://github.com/perokvist/event-driven-architecture/).

## Introduction

Follow-up to [Practial experiences with Microservices in the cloud](https://www.slideshare.net/Perkvist1/practical-experiences-with-microservices-in-the-cloud).

---

In this article we'll explore EDA mainly through "Event-based State Transfer", see [Martin Fowler's event patterns](https://martinfowler.com/videos.html#many-meanings-event).
In the examples services and compontents will collaborate through events. The service and component definition we're using is as follows.

## Service / Component

![Service definition](assets/service.png)

[Source of inspiration](http://media.abdullin.com/blog/2015/2015-03-18-edd-eBay-Barcelona.pdf#page=23)

Each service can react to events, handle request/response and emit events.
Requests can be commands or queries. Responses can be only status codes or results, the request/response could by synchronous or asynchronous, eventual consistent or not. The definition stays the same and these options are implementation details.

## Logs

This article will also look at integration between services using logs.

![Log infrastructure options](assets/logs.png)

Logging is characterized as **append only**, **ordered events** (in partition) and **offsets**, and could easily be compared to a file(s). Pushing position and acknowledging/"checkpointing" to the clients, removing broker concepts, centralized subscriptions and deadletter etc.

Using logs as integration or/and as a form of persistant model introduces new patterns when working with events. These options are going to be explored further in this article.

![integration through log](assets/service_log_integration.png)

## 1. 2PC

A common challenge when emitting changes/events is to drive the local state and publish the events in a "safe" manner. If the service updates local state first in one transaction then writes the events to a log or a broker in another transaction, you have to deal with two transactions. So you need a [two-phase commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol).
Not dealing with this could mean loosing data/events.

![Two-phase commit](assets/2pc.png)

When both transactions complete we can return 200 OK to clients. If this has business implications we would like to remove this complexity.

## 2. Polling

Another variant is to poll the source of events. This allows local state to be written, but the producing service doesn't publish the events. Instead the consumer polls based on time or offset. The consumer keeps track of the time or offset that is last polled from the producer. This variant introduces coupling between producer and consumer, as well as potential global offset for the producer, and increased traffic.

![Polling](assets/polling.png)

## 3. Single publisher

This variant has a lot in common with polling. Here a publisher does the polling and then publishes the events. But due to the fact that the publisher needs to keep track of the offset and publish the events, two-phase commit challenges come back and the implication is often that the same event could be published twice.


![function or lambda as polling publisher](assets/publisher.png)

## 4. Log subscription

In this variant we only write to the log, and then read the log to update local state. This way we only have one transaction when writing to the log.
This have similarities with using logs to avoid dual writes as detailed in [kafka dual writes](https://www.confluent.io/blog/using-logs-to-build-a-solid-data-infrastructure-or-why-dual-writes-are-a-bad-idea/).


![integration through log](assets/service_log_integration.png)

This variant also has a lot of tweaks how we could respond to commands from http requests. We could hide the async implementation or embrace it.


![response options](assets/response_options.png)

### 4.1 
We could expose our async implementation, by responding 204 when we receive the command.
To guarantee processing, we need to persit (queue) the command, or send 204 after appending to the log.

### 4.2
We could return 200 ok (or 204), when we have written to the log.

### 4.3 
We could wait and return 200 ok when we have updated the local state. This also gives us an option to return the result.

### Concurrency
Responding to the client is one thing, but what about multiple commands targeting the same instance? This also is an implementation detail, if the use case needs it, we could use locks. In the response scenario(4.3) above we use the same lock to determine when we complete.

## 5. The truth is the log. The database is a cache

A quote from Pat Helland's paper ["Immutability Changes Everything"](http://cidrdb.org/cidr2015/Papers/CIDR15_Paper16.pdf) and exemplified in ie. ["From Microliths To Microsystems"](https://www.slideshare.net/jboner/from-microliths-to-microsystems).

The final variant is to treat the log as our database. This could be done using "infinite" retention (Kafka only) or some form of snapshoting, preferably [Log Compaction](https://www.linkedin.com/pulse/kafka-architecture-log-compaction-jean-paul-azar) (Kafka only).

Then all other representations of the current state are views/projections or cache of the current state.

### Idempotency

Due to the fact that events are ["Event-based State Transfer"](https://martinfowler.com/videos.html#many-meanings-event) events, not [CRDT](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) events and common infrastructure is at-least-once delivery, idempotency on the consumer side becomes important as do order.

Some of the variants above also push "problems" to the consumer side, like updating local state and manage checkpointing. This could also introduce 2PC, but in worse case, handle/recive the same event more than once.

### Scaling writes

All scenarios above assumes that "local state" is on one node. When scaling so sets of instance state is located on different nodes, incomming traffic needs to be delegated to the node that owns the target state. The partitioning could then also be shared by the log, but doens't need to (if filtered).
Some variant of 4.X would not be suitable when scaling writes.

## Event sourcing

If local state is persisted, how is an implementation detail. Keeping a stream of events per instance (aggregate) as the source of state is often refered to as Event Sourcing. Often tied to using Domain driven design (DDD). DDD is not required for the patterns above but might be a good fit, same applies to event sourcing.

<script src="https://gist.github.com/gregoryyoung/a3e69ed58ae066b91f1b.js"></script>
Outside of DDD, this could be refered to as a journal or log. The terms journal, log and streams are found in both eventsourcing and stream processing (logs).

## Logs - bigger picture

Using logs for integration or backbone for your data platform has been described in many different ways.  Fred George described it as, ["Rapids, Rivers and Ponds"](https://vimeo.com/79866979). When all events are published to the rapids, contexts or services could subscripe and filter events through rivers. Local state or storage of filtered events becomes local ponds.

Event collaboration over different context trough a [backbone of events](https://www.confluent.io/blog/build-services-backbone-events/).

Looking back at our service/component defenition and compare that to one case of stream processing, where we have a consumer reading from one stream and publishing result on "another"(possible the same) stream.

![stream processing - enricher](assets/enricher.png)

In this scenario we see that consuming events producing new event are the same.
This could be an [enricher](http://www.enterpriseintegrationpatterns.com/patterns/messaging/DataEnricher.html) scenario.

Kafka being the log with the largest community and tooling around it, has some utils like [Kafka streams](https://balamaci.ro/kafka-streams-for-stream-processing/) making consuming events easier.

### Integration through logs

One other way of integration with logs, is to publish chanes from a database as events. Turning the database insideout. This enables supscription of change though CRUD events. And example of this is - [Bottled Water: Real-time integration of PostgreSQL and Kafka](https://www.confluent.io/blog/bottled-water-real-time-integration-of-postgresql-and-kafka/)

This could inspire a variant when an ORM publishes changes (possible 2PC).
![publishing changes through db or orm](assets/db_integration.png)

Some drawbacks of this integration style is, schema leakage, missing intent and pushing some business rules to consumers.

----

## Modelling

When collaborating through the use of events, events become the starting point for discussion and modelling. [Event storming](http://eventstorming.com/) is a way of driving your design form events, in a non techinical manner.

This will also aid you in finding the bouderies for service/components (context) (out of scope of this article).

It one thing to start fresh, but learning your domain is constant learning, finding the best feedback loops. When iterating over your contexts, things will change and version our events comes in plan - [Versioning in an Event Sourced System](https://leanpub.com/esversioning), [(video) the elephant in the room](https://skillsmatter.com/skillscasts/9652-the-elephant-in-the-room).

Modelling - [Top domain model](https://blog.scooletz.com/tag/top-domain-model/)


## Implementation

### Tactical patterns

When practicing DDD, tacitacal patterns could be utilized in implementation in the solution space. (problem space / solution space).
These patterns could be used in our service/component defenition.

![Service defentition](assets/service.png)

In addition to the basic patterns, there are a few building blocks that could also aid us.

| Name  | Formula |
| ------------- | ------------- |
| Application Service  | command -> unit  |
| Application Service  | command -> event*  |
| Enricher  | event -> event  |
| Policy  | event -> event  |
| Receptor  | event -> command  |

Examples as follows.

## Code

Commands and events are central in all examples. In OO some registry for handlers could be handy (or pattern matching available). Commands have one target, events are broadcasted.
In the *EventProcessor* we use locks as mentioned in the variants above.

### Command Dispatcher

<script src="https://gist.github.com/gregoryyoung/7677671.js"></script>
[8 lines of code - video](https://www.infoq.com/presentations/8-lines-code-refactoring)
<script src="https://gist.github.com/perokvist/2310c6f7a2bc2c16b86332903e369899.js"></script>
### EventProcessor

<script src="https://gist.github.com/perokvist/ef866f886df25d93ef7e9cca283456c0.js"></script>

### Application service
When dispaching commands to application services, an util could come i handy to support some of the variants above. This could be used in Application services for your use cases or in simple scenarios directly in the "dispatcher".
<script src="https://gist.github.com/perokvist/409f474559f44657e8d2cdf19a53b94d.js"></script>

### Test
If state is based one events, test could become; Given (past events), When (action/command) - Then (events), see [event driven verification](https://abdullin.com/sku-vault/event-driven-verification/)





# [Designing Data-Intensive Applicaitons](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications)

- [Part 1: Foundations of Data Systems](#part-1-foundations-of-data-systems)
    - [Reliable, Scalable, and Maintainable Applications](#reliable-scalable-and-maintainable-applications)
        - [Reliability](#reliability)
        - [Scalability](#scalability)
        - [Maintianability](#maintainability)
    - [Data Models and Query Languages](#data-models-and-query-languages)

# Part 1.  Foundations of Data Systems

## Reliable, Scalable, and Maintainable Applications

Data-intensive applications are built from standard building blocks that provide
common functionality.  Many applications need to:
* Store data (_databases_)
* Remember expensive operations (_caches_)
* Search by keyword/filtering (_search indexes_)
* Sending messages to another process, handle async (_stream processing_)
* Calculate large quantities of data (_batch processing_)

All of the above can be categorized under the term _data systems_. 

In addition, this book focuses on three concerns that are important in most 
software systems.

* **Reliability** - to work correctly in the face of adversity.
* **Scalability** - dealing with the growth of a system.
* **Maintainability** - Facilitate productive work on a system as engineers come and go.

### Reliability

> _Reliability_ means making systems work correctly even when faults occur.
Faults can be categorized being in **hardware**, **software**, or **humans**.
Fault-tolerance techniques can hide certain types of faults from the end user.

Expectations of reliable software:
* Application performs the function as expected by the user.
* Tolerant of unexpected ways the app is interacted with.
* Performance is good enough for the use case, under expected load and volume
* Prevents unauthorized access and abuse.

Things that go wrong in a system are called _faults_; systems that anticipate faults and 
can cope with them are called _fault-tolerant_ or _resilient_.

_Faults_ can be further defined as one component of the system deviating from its spec,
whereas _failures_ are when a system as a whole stops operating.

Most commonly, we prefer tolerating faults over preventing them, as it is difficult to 
make a system tolerant of every kind of fault.

* **Hardware Faults** - Traditionally, hardware redundancy was one of the key aids
in reducing hardware faults as single machines were commonplace.  As data size and
computing demands increase, applications began using large numbers of machines, 
preferring elasticity and flexibility over signle-machine reliability.
* **Software Errors** - Systematic errors within the system, are harder to anticipate.
Often lay dormant until triggered by an unusual set of circumstances.
* **Human Errors** - Humans are considered unreliable, config errors are the leading
cause of outages whereas hardware faults only constituted 10-25% of outages. Systems can
be made more reliable by applying the following:
    - Design systems to minimize error opportunities. i.e. abstractions, apis, and
    admin UIs that promote doing the 'right thing' and discouraging the 'wrong thing'
    - Decouple common places where mistakes are made from places where they can cause
    failures. i.e. sandbox nonprod environments
    - Unit tests, whole system integration tests, manual tests
    - Allow quick recovery - make rollbacks fast. i.e. fast config rollbacks, 
    rolling out new code gradually, provide tools to recompute data
    - Detailed and clear monitoring, telemetry, performance metrics

### Scalability

> _Scalability_ means having strategies for keeping performance good when under load.

It is also crucial to ask questions like "If the system grows in a particular way, 
what are our options for coping with the growth?"

#### Describing Load 

Load can be described with a few numbers, named _load parameters_.  Best parameter 
choice varies based on the architecture of the system (i.e. reqs/ to web server, 
ratio of reads and writes to a database, hit rate on cache).

#### Twitter Example

Two of twitters main operations:
* Post tweet: User can publish a new message to their followers (4.6k request/s
on average, over 12k requests/sec at peak).
* Home timeline: A user can view tweets posted by the people they follow (300k request/sec)

The main proplem isn't due to writes, rather doe to _fan-out_.

Ways of implementing these operations:
1. Posting a tweet inserts the new tweet into a global collection of tweets. When
a user request their home timeline, look up all the people they follow, find all the 
tweets for each of those users, and merge them (sorted by time). In a relational
database, you could write a query like:
```sql
SELECT tweets.*, users.* FROM tweets
    JOIN users   ON tweets.sender_id    = users.id
    JOIN follows ON follows.followee_id = users.id
    WHERE follows.follower_id = current_user
```
2. Maintain a cache for each user's home timeline - like a mailbox of tweets for
each recipient user.  When a user posts a tweet, look up all people that follow that
user and insert the new tweet into each of their home timeline caches (_fan-out_).

The first version of Twitter used approach 1, but the system struggled to keep 
up with load from home timeline queries and switched to approach 2.  This is because
the rate of published tweets is two orders of magnitude lower than rate of home
timeline reads.

But, due to high-follower users (possible 30M writes per tweet), Twitter now uses 
a hybrid approach, where high-follower user tweets are instead queried separately 
on home timeline request because of high write cost to home timelines.

#### Describing Performance

After describing load, you can investigate what happens when load increases:

* When a load parameter is increased and system resources are kept the same, how
is system performance affected?
* When you increase a load parameter, how much do you need to increase the resources
to keep performance unchanged?

> ##### Latency and response time
> _Latency_ and _response time_ are not the same.  Response time is what the client sees,
including the actual time to process the request, network delays, and queueing delays.
Latency is the duration that a request is waiting to be handled.

The average response time, or arithmetic mean, of a service is commonly reported. 
This isn't a good metric as it doesn't show how many users are actually affected.

#### Percentiles

- Using percentiles is a better approach.  The median(_50th percentile, p50_) is 
a good metric because you can see that 50% of users are served in less than the 
median response time.
- To determine bad outliers you can look at higher percentiles: 95th, 99th, and
99.9th percentiles are common (abbreviated as p95, p99, and p999)

High percentiles of response times, _tail latencies_, are important because they
directly affect users' experience of the service.

Percentiles are often used in _service level objectives_ (SLOs) and _service
level agreements_ (SLAs), contracts that define the expected performance and 
availability of a service.
> An SLA may state that a service is considered to be up if it has a median response
time of lest than 200ms and <1s p99.

Queuing delays often account for a large part of the response time at high percentiles.
Due to this, it is crucial to measure response time on the client side.

> ##### Percentiles in Practice
> High percentiles are important when backend services are called many times 
when completing a single end-user request.  Even if calls are made in parallel,
theresponse time is only as fast as the longest call.
> _tail latency amplification_ - Even if only a small number of backend calls are slow, 
the chance of gettinga  slow call increases if the end-user request require multiple
backend calls, such that a higher proportion of end-user requests end up being slow.

#### Approaches for Coping with Load

How do we maintain good performance even when our load parameters increase by some
amount?

* _Scaling up_: Moving to a more powerful machine.
* _Scaling out_: distributing the load across multiple smaller machines.  Also
known as a _shared-nothing architecture_.

The choices on how to design the architecture of a system is multi-faceted.  There's
no such thing as a generic, one-size-fits-all scalable architecture.

Even though they are specific to a particular application, scalable architecutres are
usually built from general-purpose buliding blocks, arranged in familiar patterns.

### Maintainability

> _Maintainability_ is about making life better for the engineering and operations
teams who need to work with the system. Good abstractions can help reduce complexity
and make the system easier to modify and adapt for new use-cases.

Software should be designed in such a way that it minimizes pain during maintenance.
There are three design principles for software systems to pay attention to
* **Operability**. Make it easy for operations teams to keep the system running 
smoothly.
* **Simplicity**. Make systems easy to understand by removing unecessary complexity.
* **Evolvability**. Make it easy for the system to be changed in the future.

#### Operability: Making Life Easy for Operations

A good operations team is responsible for the following:
* Monitoring system health and service restoration if its in a bad state
* Tracking down the cause of problems, i.e. system failures
* Keeping software and platforms up to date
* Understand dependencies between systems
* Anticipate future problems and prevent them before they occur

#### Simplicity: Managing Complexity

There are various symptoms of complexity, including:
* Explosion of the statue space
* Tight coupling of modules
* Tangled dependencies 

Making a system simpler does not necesarily mean reducing its functionality; it
can also mean removing _accidental_ complexity.

> Complexity can be defined as _accidental_ when it is not inherent in the problem
that to software solves.

One of the best tools we have for removing accidental complexity is _abstraction_.
A good abstraction can hide a great deal of implementation detail behind a clean,
easy to understand facade.

#### Evolvability: Making Change Easy

In terms of organization processes, _Agile_ working patterns provide a framework for
adapting to change. 

The easy with which you can modify a data system is closely linked to its simplicity
and abstractions. Simple and easy to understand systems are typically easier to 
modify.

## Data Models and Query Languages




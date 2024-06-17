# [Designing Data-Intensive Applicaitons](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications)

- [Part 1: Foundations of Data Systems](#part-1-foundations-of-data-systems)
    - [Reliable, Scalable, and Maintainable Applications](#reliable-scalable-and-maintainable-applications)
        - [Reliability](#reliability)
        - [Scalability](#scalability)
        - [Maintianability](#maintainability)
    - [Data Models and Query Languages](#data-models-and-query-languages)
        - [Relational Model Versus Document Model](#relational-model-versus-document-model)
        - [Query Languages for Data](#query-languages-for-data)

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

Data models are one of the most important parts of developing software.  
They affect how software is written as well as how we think about the problem
that we are solving.

### Relational Model Versus Document Model

One of the best known data modesl today is SQL, where data is organized into
*relations* (tables in SQL), where each relation is an unordered collection of 
*tuples*.

#### The Birth of NoSQL

The term 'NoSQL' doesn't refer to any specific technology, but there are several 
reasons as to why NoSQL databases have seen wide adoption:
* Scalability greater than relational databases, large datasets and high write throughput.
* Preference of open source technology over commercial
* Specialized query operations
* Less restrictive than relational schema

_Polyglot persistence_. Relational databses will continued to be used alongside
a broad variety of nonrelational datastores.

#### The Object-Relational Mismatch

Due to many applications being developed using object-oriented programming
languages, there is an awkward translation layer that is required when using the
SQL data model between the application code and the database model of tables,
rows, and columns. This disconnect is somethines called an _impedence mismatch_.

Object-relational mapping frameworks (ActiveRecord, Hibernate) reduce boilerplate
code but cannot completely hide the difference between the two models.

One-to-many relationships can be described in a resume example: people may have
more than one job in their career, and people may have varying numbers of education.
This can be represented in the following ways
* Put positions and education in separate tables and reference via foreign key to
the **user** table
* Structured data types and XML data; allowing multi-valued data stored in a single
row.
* Encode jobs and education as JSON or XML document, and store on a text column.

Data structures like resumes can also be represented with a self-contained
*document*, like encoding in JSON. This can reduce impedance mismatch between
application code and the storage layer because of the lack of a schema.

The JSON/document model also has better _locality_, meaning that instead of 
performing multiple queries to fetch data, all of the data is in one place.

One-to-many relationships tend to imply a tree structure.

#### Many-to-One and Many-to-Many Relationships

It can be advantageous to have standardized lists for properties like geographic
regions and industries:
* Consistent style/spelling
* Avoids ambiguity
* Ease of updating as a value is stored in one place
* Localization support, region/industry can be displayed in viewer's language
* Better search

Using IDs is advantagous as they don't have any meaning to humans. Therefore,
it never needs to change even if the information it references changes. If 
the information that changes is duplicated, it needs to be updated everywhere 
it is used.  Removing duplication is the key idea behind _normalization_ in databases.

_Normalizing_ data requires _many-to-one_ relationships, which is difficult to
fit into the document model. But, fits very well with the relational model - it
is normal to refer to rows in other tables by ID, because joins are easy.

**Many-to-Many relationships**. Many records in one table relate to many records 
in another table.
Can be described as many users on LinkedIn can be employed by an organization, 
and an organization can have many employees

#### Are Document Databases Repeating History?

The post popular database for business data processing in the 1970s was IBM's 
Information Management System (IMS).

IMS used a _hierarchical model_, representing all data as a tree of records nested
within records. It was good for one-to-many relationships, but many-to-many was
difficult and it did not support joins

##### The network model

Standardized by a committee called the Conference on Data Systems Languages
(CODASYL); also known as the _CODASYL model_.

Generalized the hierarchical model, in the tree structure - every record could
have multiple parents.  This allowed many-to-one and many-to-many relationships
to be modeled

The links between records were similar to pointers in a programming language. 
The only way of getting a record was following a path form the root record called
an _access path_.

A query in CODASYL was performed via moving a cursor through the database by 
iterating over lists of records. This was difficult to make changes to and
if you didn't have the path you wanted, it was difficult to find the record.

##### The relational model

Layed out all data in the open: a relation (table) is a collection of tuples.
You can read any and all rows in a table, selecting those that match an 
arbitrary condition.

Query optimizers decide which parts of the query to execute in which order, 
and what indexes to use. 

The relational model made it much easer to add new features to applications.

##### Comparison to document databases

Document databases reverted back to the hierarchical model in one way, storing
nested records within a parent record rather than a separate table.

When representing one-to-many and many-to-many relationships, relational and
document databases both use unique identifiers for a related item.  These
are _foreign keys_ in relational model and _document references_ in the 
document model.

#### Relational Versus Document Databases Today

The main argument for document models are because of their schema flexibility,
better performance (locality), and similarity to data structures used
by the appliation. The relational model provides better join support, and
many-to-one and many-to-many relationships.

##### Which data model leads to simler application code?

If using document like structures (i.e. tree of one-to-many relationships), then
the document model is better.  The relational technique of _shredding_ - splitting
a document like structure into multiple tables can lead to cumbersome schemas
and convoluted application code.

Limitations of the document model include not being able to directly reference
nested items in a document, but this usually isn't a problem if documents
aren't too deeply nested.

Poor support for joins are another factor against using the document model - for
example many-to-many relationships may never be needed in an analytics app that 
records what events occur at which time.

If many-to-many relationships are used, this can result in significantly more 
complex application code and worse performanceif the document model is used.

To summarize, for highly interconnected data, the document model is awkward,
the relational model is acceptable, and graph models are the most natural.

##### Schema flexibility in the document model

Most document databases do not enforce a schema, which means arbitrary keys and 
values can be added to a document.

Document databases are considered _schema-on-read_.
* **Schema-on-read**. The structure of the data is implicit, and only interpreted
when the data is read.
* **Schema-on-write**. The traditional approach of relational databases, where
the schema is explicit and the database ensures all written data conforms to it.

Schema-on-read is similar to dynamic (runtime) type checking in programming 
languages, whereas schema-on-write is similar to static (compile-time) type
checking.

In a document database, you would just start writing new documents with a new 
field and have code account for cases of old documents:
```javascript
if (user && user.name && !user.first_name) {
    // Documents written before Dec 8, 2013 don't have first_name
    user.first_name = user.name.split(" ")[0];
}
```

On the other hand, a schema-on-write implementation performs a migration such as:
```sql
ALTER TABLE users ADD COLUMN first_name text;
UPDATE users SET first_name = split_part(name, ' ', 1);     -- PostgreSQL
UPDATE users SET first_name = substring_index(name, ' ', 1);     -- MySQL
```

Most RDBMSes run ALTER TABLE in a few milleseconds. MySQL is an exception as it
copies the entire table on ALTER TABLE, which can result in minutes or hours of
downtime on larger tables.

Running UPDATE on any large table is likely to be slow on any database as each
row needs to be rewritten.

Schema-on-read can be advantageous if the items in the collection don't have 
the same structure for some reason (heterogeneous data), because:
* Many different types of objects
* Data structure is determined by an external system that is subject to change

##### Data locality for queries

Documents are usually a continuous string - if your application often needs
the entire document, then there is a performance advantage to this _storage locality_.

It is generally recommended to keep documents small because:
* Databases load the entire document even if you only need a small portion.
* Updates to a document usually require the entire document to be rewritten.

##### Convergence of document and relational databases

If a database is able to handle document-like data and perform relational 
queries on it, applications can use the combination of features that best fits 
their needs.

A hybrid of relational and document models is a good route for databases to 
take in the future.

### Query Languages for Data

SQL is a _declarative_ query language, where as IMS and CODASYL queried the 
data base using _imperative_ code.

Many commonly used programming languages are _imperative_:

```javascript
function getSharks() {
    var sharks = [];
    for (var i = 0; i < animals.length; i++) {
        if (animals[i].family === "Sharks") {
            sharks.push(animals[i]);
        }
    }
    return sharks;
}
```

Here is the same query in SQL:
```sql
SELECT * FROM animals WHERE family = 'Sharks';
```

An imperative language tells the computer to perform certain operations in a
certain order.

In a declarative query language, you just specify the pattern of the data you
want - what conditions the results must meet, and how you want the data to be
transformed - but not how to achieve that goal.  It is up to the database 
system's query optimizer to decide which indexes and which join methods to use,
and what order to execute the query in.

Declarative languages lend themselves to parallel execution because they only
specify the pattern of the results, not the algorithm that is used to determine 
the results.

#### Delcarative Queries on the Web

CSS is considered a declarative styling in the web browser and is much better
than manipulating styles imperatively in JavaScript.

#### MapReduce Querying

_MapReduce_ is a programming model for procesing large amounts of data in bulk
across many machines.

MapReduce is neither a declarative language or imperative query API, but 
somewhere in between: the logic of the query is expressed with snippets of code,
which are called repeatedly by the processing framework. It is based on the 
**map** and **reduce** functions that exist in many functional programming
languages.

Here's an example where you add an observation record for every animal seen
and you want to see how many sharks you have sighted per month:

```sql
SELECT date_truc('month', observation_timestamp) AS observation_month,
       sum(num_animals) AS total_animals
FROM observations
WHERE family = 'Sharks'
GROUP BY observation_month;
```

The same can be expressed with MongoDB's MapReduce feature:

```javascript
db.observations.mapReduce(
    function map() {
        var year = this.observationTimestamp.getFullYear();
        var month = this.observationTimestamp.getMonth() + 1;
        emit(year + "-" + month, this.numAnimals);
    },
    function reduce(key, values) {
        return Array.sum(values);
    },
    {
        query: { family: "Sharks" },
        out: "monthlySharkReport"
    }
)
```

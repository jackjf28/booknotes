# [Designing Data-Intensive Applicaitons](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications)

- [Part 1: Foundations of Data Systems](#part-1-foundations-of-data-systems)
    - [Reliable, Scalable, and Maintainable Applications](#reliable-scalable-and-maintainable-applications)
        - [Reliability](#reliability)
        - [Scalability](#scalability)
        - [Maintianability](#maintainability)
    - [Data Models and Query Languages](#data-models-and-query-languages)
        - [Relational Model Versus Document Model](#relational-model-versus-document-model)
        - [Query Languages for Data](#query-languages-for-data)
        - [Graph-Like Data Models](#graph-like-data-models)
    - [Storage and Retrieval](#storage-and-retrieval)
        - [Data Structures That Power Your Database](#data-structures-that-power-your-database)
        - [Transaction Processing or Analytics?](#transaction-processing-or-analytics)
        - [Column-Oriented Storage](#column-oriented-storage)
    - [Encoding and Evolution](#encoding-and-evolution)
        - [Formats for Encoding Data](#formats-for-encoding-data)
        - [Modes of Dataflow](#modes-of-dataflow)
- [Part 2: Distributed Data](#part-2-distributed-data)
    - [Replication](#replication)
        - [Leaders and Followers](#leaders-and-followers)
        - [Problems with Replication Lag](#problems-with-replication-lag)
        - [Multi-Leader Replication](#multi-leader-replication)
        - [Leaderless Replication](#leaderless-replication)

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

For example, say the observations collection contains these two documents:

```json
{
    observationTimestamp: Date.parse("Mon, 25 Dec 1995 12:34:56 GMT"),
    family:     "Sharks",
    species:    "Carcharodon carcharias",
    numAnimals: 3,
}
{
    observationTimestamp: Date.parse("Tue, 12 Dec 1995 16:17:18 GMT"),
    family:     "Sharks",
    species:    "Carcharodon taurus",
    numAnimals: 4,
}
```

The `map` function would be called once for each document, resulting in 
`emit("1995-12", 3)` and `emit("1995-12, 4)`. Then the `reduce` function would
be called with `reduce("1995-12", [3,4])` returning 7.

The `map` and `reduce` functions are limited in what they're allowed to do.
They can only use the data that is passed to them as input, so they cannot
perform additional queries, and must not have any side effects.

### Graph-Like Data Models

If many-to-many relationships are very common in your data, it becomes more natural
to start modeling your data as a graph.

Graphs consist of two kinds of objects: _vertices_ (i.e. _nodes_ or _entities_)
and _edges_ (i.e. _relationships_ or _arcs_). Some of the typical examples of
graph data models include:
* **Social Graphs**. Vertices are people, edges indicate who knows eachother
* **The web graph**. Vertices are web pages, edges indicate HTML links
* **Road or rail networks**. Vertices are junctions, and edges represent railways
or roads between them.

Usage of graphs is to provide a consistent way of storing completely different
types of objects in a single datastore.

#### Property Graphs

In the property graph model, each vertex consists of:
* A unique identifier
* A set of outgoing edges
* A set of incoming edges
* A collection of properties (key-value pairs)

Each edge consists of:
* A unique identifier
* The vertex at which the edge starts (the _tail vertex_)
* The vertex at which the edge ends (the _head vertex_)
* A label to describe the kind of relationship between the two vertices
* A collection of properties (key-value pairs)

You can think of a graph store as two relational tables, one for vertices
and one for edges.

Property graph using relational schema:
```sql
CREATE TABLE vertices (
    vertex_id       integer PRIMARY KEY,
    properties      json
);
CREATE TABLE edges (
    edge_id         integer PRIMARY KEY,
    tail_vertex     integer REFERENCES vertices (vertex_id),
    head_vertex     integer REFERENCES vertices (vertex_id),
    label           text,
    properties      json
);

CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```

Important aspects of this model:
* Any vertex can have an edge connecting it with any other vertex. No schema
restrictions.
* Given any vertex, you can efficiently find incoming and outgoing edges, thus
_traverse_ the graph
* By using different labels for different relationships, you can store several
different kinds of information in a single graph

The above features give graphs a great deal of flexibility for data modeling.
For example, regional structures in different countries may be difficult to
express in traditional relational schema, where as graphs can accomodate this
well.

Graphs are good for evolvability; as you add features to your application,
a graph can easily be extended to accommodate changes in your application's
data structures.

#### The Cypher Query Language

**Cypher** is a declarative query language for property graphs, created for the
Neo4j graph database.

The below example shows the Cypher query to insert into a graph database.
```sql
CREATE
    (NAmerica:Location {name:'North America', type:'continent'}),
    (USA:Location      {name:'United States', type:'country'}),
    (Idhao:Location    {name:'Idaho',         type:'state'}),
    (Lucy:Person       {name:'Lucy' }),
    (Idaho) -[:WITHIN]->  (USA)  -[:WITHIN]-> (NAmerica),
    (Lucy)  -[:BORN_IN]-> (Idaho)
```

An example query in Cypher would be to find all the vertices that have a `BORN_IN`
edge to a location within the US and a `LIVING_IN` edge to a location within
Europe, and return the `name` property for each of these vertices.

Example of this query in Cypher:

```sql
MATCH
    (person) -[:BORN_IN]->  () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
    (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'}),
RETURN person.name
```

The above query can be read as:

1. Find any vertex (call it `person`) that meets _both_ of the following conditions
2. `person` has an outgoing `BORN_IN` edge to some vertex. From that vertex, you
can follow a chain of outgoing `WITHIN` edges until eventually you reach a vertex
of type `Location`, whose name property is equal to "United States"
3. That same `person` vertex also has an outgoing `LIVES_IN`edge. Following 
that edge, and then a chain of outgoing `WITHIN` edges, you eventually reach 
a vertex of type `Location`, whose `name` property is equal to "Europe".
4. For each such `person` vertex, return the `name` property.

#### Graph Queries in SQL

SQL _can_ be used to query graph data, but with some difficulty.  In a relational
database, you usually know in advance which joins need to be queried.  In a graph
query, you may need to traverse a variable number of edges before you find the 
vertex that you're looking for - the number of joins is not fixed in advance.

Variable-length traversal paths in a query can be expressed in SQL using something
called _recursive common table expressions_ (the WITH RECURSIVE syntax). It is
very lengthy and clumsy in comparison to the 4 lines of Cypher to do effectively
the same thing.

#### Triple-Stores and SPARQL

Triple-stores and the property graph model are mostly equivalent, but a triple-store
has all information stored in the form of a very simple three-part statement:
(_subject, predicate, object)_. e.g. (_Jim, likes, bananas)_.

The subject is equivalent to the vertex in a graph, object is one of two things:
1. A value in a primitive datatype. (e.g. (_lucy, age, 33_) or vertex `lucy` with properties `{"age": 33}`)
2. Another vertex in the graph. In this case, the predicate is an edge between 
the two, where the subject is the tail vertex and the object is the head.

Here is an example of triples in a format called _Turtle_:

```turtle
@prefix : <urn:example:>.
_:lucy      a       :Person.
_:lucy      :name   "Lucy".
_:lucy      :bornIn _:idaho.
_:idaho     a       :Location.
_:idaho     :name   "Idaho".
_:idaho     :type   "state".
_:idaho     :within _:usa.
.
.
.
```

##### The semantic web

The idea that websites already publish information as txt and pictures for humans,
so why don't they also poblish information as machine-readable data for computers to read?

The _Resource Description Framework_ (RDF) was intended as a mechanism for 
different websites to publish data in a consistent format, allowing data
from different websites to be automatically combined into a _web of data_.

##### The RDF data model

The _Turtle_ language is a human-readable formate for RDF data.  RDF is sometimes
written in XML as well.

It is also designed for internet-wide data exchange

##### The SPARQL query language

_SPARQL Protocol and RDF Query Language_ (SPARQL) is a query language for 
triple-stores using the RDF data model. Predates Cypher and Cypher's pattern
matching is borrowed from SPARQL.

The above Cypher query in SPARQL format:

```sparql
PREFIX: <urn:example>

SELECT ?personName WHERE {
    ?person :name ?personName.
    ?person :bornIn  / :within* / :name "United States".
    ?person :livesIn / :within* / :name "Europe".
}
```

#### The Foundation: Datalog

_Datalog_ is a much older language than SPARQL and Cypher, citing back to the
1980s.

Datalog's data model is similar to triple-store, but instead of (_subject, predicate, object_),
we write it as _predicate(subject, object)_.

```
name(namerica, 'North America')
type(namerica, continent)

name(usa, 'United States')
type(usa, country)
within(usa, namerica)

name(idaho, 'Idaho')
type(idaho, state)
within(idaho, usa)

name(lucy, 'Lucy')
born_in(lucy, idaho)
```

Example query from the BORN_IN/LIVES_IN Cypher query:
```
within_recursive(Location, Name) :- name(Location, Name).       /* Rule 1 */

within_recursive(Location, Name) :- within(Location, Via),      /* Rule 2 */
                                    within_recursive(Via, Name).

migrated(Name, BornIn, LivingIn) :- name(Person, Name),         /* Rule 3 */
                                    born_in(Person, BornLoc),
                                    within_recursive(BornLoc, BornIn),
                                    lives_in(Person, LivingLoc),
                                    within_recursive(LivingLoc, LivingIn).

?- migrated(Who, 'United States', 'Europe');
/* Who = 'Lucy' */
```

Datalog queries take small steps at a time rather than Cypher and SPARQL's usage
of SELECT.  Rules are defined and can refer to one another.  With this, complex
queries can be built up a small piece at a time.


## Storage and Retrieval

This chapter covers the question of how can we store the data that we're given,
and how can we find it again when we're asked for it.

### Data Structures That Power Your Database

Consider this simple database:

```bash
db_set () {
    echo "$1,$2" >> database
}
db_get () {
    grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

These functions implement a key-value store.  The key and the value can be almost
any value.  Every call to `db_set` appends to the end of a file.  Old values 
aren't overwritten, you simply look at the last value for the most recent value
(`tail -n 1` returns the last item in the collection).

`db_set` has pretty good performance due to its append-only nature, many 
databases use a similar append-only data file called a _log_.

> _Log_. An append-only sequence of records. Optionally human-readable; it might
be binary and intended only for other programs to read.

Lookups have bad performance on logs at O(n).  An _index_ is needed to 
efficiently find a value for a particular key.

_Indexes_ are an _additional_ structure that are derived from the primary data.
They usually slow down write speed, because the index needs to be updated every
time the database is written to.

_Indexes_ are an important tradeoff, though: well-chosen indexes speed up read
queries, but slow down writes.

#### Hash Indexes

Key-value stores are similar to the dictionary type in many programming 
languages, typically a hash map.

A simple indexing strategy is to keep an in-memory hash map where every key is
mapped to a byte offset in the data file.  Whenever a new key-value pair is 
appended to the file, you update the hash map to reflect the offset of the data
you wrote.  Looking up involves using the hash map to find the offset, seek
the location, and read the value.

How do we avoid running out of disk space in append only files?

We can break the log into segments of a certain size when a segment file 
reaches a certain size, and making subsequent writes to a new segment file.
Then, _compaction_ can be performed. _Compaction_ means throwing away duplicate 
keys in a log, and keeping only the most recent update for a key.

Because compation makes segments smaller, segments can also be merged together 
at the same time as performing compaction. Segments are append-only, so a new
file is always created for the merged segment.

Read requests are then updated to use the new merged segment, and old segment
files that were merged can be deleted.

Some issues to consider with this implementation:
* File format: Binary that first encodes the string length in bytes is faster 
and simpler to use.
* Deleting records: When deleting a key-value pair, you can append a deletion
record (_tombstone_), when log segments merge - the tombstone tells the merge
process to delete the value for the deleted key.
* Crash recovery: To minimize hash map index loss on system restart, store
a snapshot of each segment hash map on disk .
* Partially written records: Include checksums, allowing partial writes to be
ignored.
* Concurrency control: Common implmentation is to have one writer thread. Data 
file segments are write-only, so they can be read concurrently by multiple 
threads.

Append-only design is good because of the following:
* Append and merge are sequential operations and faster than random writes.
* Concurrency/crash recovery are simpler if files are append-only.
* Merging old segments prevents fragmentation over time.

Limitations:
* Hash table must fit in memory.
* Range queries aren't efficient. 

#### SSTables and LSM-Trees

_Sorted String Tables_ (SSTables) modify the segment format: 
* Require that the sequence of key-value pairs are sorted by key. 
* Require that each key only appears once within each merged segment file.
(compaction ensures this).

SSTable advantages:
* Merging is simple and efficient. Similar to mergesort.
* No longer need to keep an index of all keys in memory.  Because the keys
are ordered, you can instead maintain a _sparse index_ of keys. One key per
every few KB of segment file is sufficient.
* Since reads need to scan over several key-value pairs in the requested range,
it's possible to group those records in a block and compress before writing
to disk.  Each entry of the sparse in-memory index then points at the start
of the compressed block

##### Constructing and maintaining SSTables

We can make the storage engine work as follows:
* When a write comes in, add to in-memory balanced tree (red-black tree). This
tree is considered a _memtable_.
* When the memtable reaches some threshold, write to disk as an SSTable file.
The new file becomes the most recent database segment.
* To read, try to find the key in the memtable, then in most recent SSTable
file and so on.
* Occasionally run a merge/compaction process

To avoid lost writes due to a crash, keep a separate log on disk where every
write is immediately appended.

##### Making an LSM-tree out of SSTables

The algorithm above is used in LevelDB and RocksDB, the _Log-Structured Merge-Tree_.
Storage engines that are based on this principle of merging and compacting 
sorted files are often called LSM storage engines.

##### Performance optimizations

In order to optimize LSM-tree algorithm key lookups, storage engines often use 
additional _Bloom filters_.

**Bloom Filter**. A memory efficient data structure for approximating the 
contents of a set. It can tell you if a key does not appear in a database,
thus saving unnecessary disk reads.

Common options for determining when to compact and merge are _size-tiered_ and
_leveled_ compaction.

**Size-tiered**. Newer and smaller SSTables are merged into older and larger 
SSTables.

**Leveled compaction**. Key range is split into smaller SSTables and older data 
is moved into separate "levels", allowing compaction to proceed more
incrementally

#### B-Trees

_B-Trees_ are the most widely used indexing structure.

They keep key-value pairs sorted by key, similar to SSTables.

In addition, B-trees break the database into fixed-size _blocks_ or _pages_ (Usually
4KB in size) and read or write one page at a time.  Each page can be identified
using either an address or location, allowing one page to reference another.

One page is considered the _root_ of the B-tree, and contains several keys and 
references to child pages.

Eventually we get down to a page containing individual keys (a _leaf page_),
which contains the value for each key inline or contains references to 
the pages where the value can be found.

**Branching Factor**. The number of references to child pages in one page of the 
B-tree.

To update a value, search for a leaf page contianing that key, change the value
in tha tpage, and write the page back to disk

To add a new key, find the page whose range encompasses the new key and add it
to the page.  If there's no free space, split into two half-full pages and
the parent page is updated to reference the two new key ranges.

##### Making B-trees reliable

Some operations require multiple pages to be overwritten, if there is a 
database crash, this can result in a corrupted index.

B-tree implementations commonly include an additional data structure on disk
called a _write-ahead log_ (WAL) to make databases resilient to crashes and
restores the B-tree back to consistent state.

WALs are an append-only file where every B-tree modification must be written
before it's actually applied.

Concurrency is also a concert. If muliple threads are to be used to access the
B-tree, _latches_ (lightweight locks) are used for protection.

##### B-tree optimizations

* Copy-on-write schemes can be used in contrast to overwiting pages/WAL.
* Key abbreviation to save on space
* Sequential ordering of leaf pages to reduce seek time, growth can cause issues.
LSM-Trees can be better for sequential reads
* Pointers between sibling leaf pages

#### Comparing B-Trees and LSM-Trees

As a rule of thumb, LSM-trees are typically faster for writes, whereas 
B-trees are faster for reads. LSM-tree reads are slower because they have
to check several different data structures and SST files during compaction.

##### Advantages of LSM-trees

B-tree indexes must write every piece of data at least twice, once to the WAL,
and once to the tree page (more if the page is split). 

**Write Amplification**. One write to the database resulting in multiple writes
to disk over the course of the database's lifetime.

LSM-trees are able to sustain higher write throughput that B-trees, as sometimes
they have lower write amplification

LSM-trees also have better compression, producing smaller files on disk.

They also have lower storage overhead, as they aren't page oriented and remove
fragmentation through SSTable rewrites.

##### Downsides of LSM-trees

The compaction process can sometimes interfere with performance of ongoing reads
and writes.

Another issue with compaction arises at high write throughput: disk bandwidth
is shared between the initial write (logging and flushing memtable to disk) and
compaction threads. This issue becomes more prominent as the database gets larger.

High write throughput and misconfigured compaction can result in compaction 
lagging behind, resulting in the number of unmerged segments growing until
we run out of disk. Explicit monitoring is needed for this scenario.

A B-tree advantage is that each key exists only once, which is good for databases
that want to offer strong transactional semantics.

#### Other Indexing Structures

It is common to have _secondary indexes_.  In relational databases, you can 
create several secondary indexes on the same table using `CREATE INDEX`, and
are crucial for efficient joins.

The main difference between primary and secondary indexes is that secondary indexes
are not unique; i.e., there might be many rows with the same key.

##### Storing values within the index

The value in a key-value index can be:
* The actual row (document,vertex)
* A reference to the row stored elsewhere, the place where rows are stored is 
known as a _heap file_.

The heap file approach is common because it avoids duplicating data where multiple
secondary indexes are present.

_Clustered index_. Storing the indexed row directly within an index.

Clustered indexes can solve the performance penalty involved with hopping from
an index to the heap file.

A compromise between clustered indexes and heap files is known as a _covering
index_ or _index with included columns_, which stores some of the table
columns within the index.

##### Multi-column indexes

Most common  type of multi-column index is a _concatenated index_, which combines
several ields into one key by appending one column to another.

A concrete example  would be a phone book, which provides an index from 
(_lastname, first-name_) to a phone number.

Multi-dimensional indexes are a general way of querying several columns at once.

Here's an example where we search for all restaurants in a specific latitude 
and longitude:

```sql
SELECT * FROM restaurants WHERE latitude  > 51.4946 AND latitude  < 51.5079
                            AND longitude > -0.1162 AND longitude < -0.1004;
```

B-trees and LSM-tree indexes are inefficient in executing this as they cannot
give both latitude and longitude simultaneously.

##### Full-text search and fuzzy indexes

Fuzzy querying allows search for _similar_ keys, such as mispelled words.

Full-text search engines can allow a search for one word to be expanded to
include synonyms, ignore grammatical variations of words, and search for occurrences
of words near each other in the same document.

##### Keeping everything in memory

As RAM becomes cheaper, it becomes more feasible to keep datasets entirely in
memory as most datasets are not _that_ big. From this has emerged the
development of _in-memory databases_.

Restarts in in-memory databases mean they need to reload their state from disk
or over network from a replica.

The performance advantage of in-memory datastures comes from avoiding overhead
of encoding in-memory data structures in a form that can be written to disk.

Products such as VoltDB, MemSQL, and Oracle TimesTen are in-memory databases.
Redis and Couchbase provide weak durability by writing to disk asynchronously.

In-memory databases also can provide data models that are difficult to implement 
in disk-based indexes.  An example of this is Redis offering a database-like 
interface to various data structures like sets and priority queues.

### Transaction Processing or Analytics?

Interactive processes between user and application constitute an access
pattern in databases known as _online transaction processing_ (OLTP).

_Data analytics_ have a very different access pattern, scanning over a huge
number of records, only reading a few columns per records, and 
calculating aggregate statistics rather than returning raw data.

Data analytic queries create a different pattern from OLTP, called _online
analytic processing_ (OLAP).

In the late 1980s and early 1990s, there was a trend for companies to stop using 
their  OLTP systems for analytics purposes, and run the analytics on a separate
database instead.  This separate database was called a _data warehouse_.

#### Data Warehousing

A _data warehouse_ is a separate database that analyst can query without 
affecting OLTP operations.  It contains a read-only copy of the data in all 
various OLTP systems in the company.  Data is extractef rom OLTP databases,
transformed into an analysis-friendly schema, cleaned up, and then loaded into
a data warehouse.

The process of getting data into the warehouse is known as _Extract-Transform-Load_
(ETL).

An advantage of using a separate data warehouse, rather than querying OLTP
systems, is that the warehouse can be optimized for analytics.

Data warehouses are commonly relational, but the internals can be much different
than OLTP systems.

Many data warehouses are used in a fairly formulaic style known as a _star schema_.

At the center of the star schema is a _fact table_.  Each row of the fact table
represents an event that occured at a particular time.  o

Some of the fact table columns are attributes, other columns are foreign key 
references to other tables, called _dimension tables_.

_Dimensions_ represent the _who, what, where, when, how, and why_ of the event.

A variation of this template is the _snowflake schema_, where dimensions are
further broken down into sub dimensions.  They are more normalized than star
schemas, but star schemas are usually preferred because they are easier to 
work with.

In a typical data warehouse, tables are often very wide.  Fact tables often have
over 100 columns.  

### Column-Oriented Storage

Although fact tables are often over 100 columns wide, typical data warehouse
queries only access 4 or 5 of them. 

Row-oriented storage engines will load all of the rows from disk to memory,
parse them, and filter out those that don't meet the required conditions.
This can take a long time.

_Column-oriented storage_'s idea is don't store all the values from one row 
together, but store all the values from each column together instead.  If each
column is stored in a separate file, a query only needs to read and parse those
columns that are used in that query, which can save a lot of work.

The column-oritented storage layout relies on each column file containing the
rows in the same order.  To reassemble an entire row, you can take the nth entry
from each of the individual column files and put them together to form the nth
row of the table.

#### Column Compression

Demands on disk throughput can be further reduces by compressing data.

One technique that is used in warehouses is _bitmap encoding_. Often, the number
of distinct values in a column is small compared to the number of rows.  We
can take a column with _n_ distinct values and turn it into _n_ separate bitmaps,
those bitmaps can be stored with one bit per row.

Bitmaps can additionally be _run-length encoded_ if the number of distinct values
is large.

One example of bitmap usage:

```sql
WHERE product_sk in (30, 68, 69);
```
Load the three bitmaps for product_sk = 30, 68, and 69, and calculate the bitwise
OR of the three bitmaps.

##### Memory bandwidth and vectorized processing

A big bottleneck for warehouse queries is the bandwidth from getting data from
disk to memory. 

Besides reducing the volume of data needed to be loaded from disk, column-oriented
storage layouts are good for efficient usage of CPU cycles. 

#### Sort Order in Column Storage

It doesn't necessarily matter which order the rows are stored. However, we can choose
to impose an order, similar to SSTables, and use this as an indexing mechanism.

The data needs to be sorted an entire row at a time on one column, a second column 
can also be used to determine a sort order of any rows with the same value in
the first column.

Sorting can also help with compression, and enable run-length encoding.

A clever extension of this idea is to store the same data sorted in several 
different ways as data needs to be replicated to multiple machines anyways.
You can then use the version of the data that best fits your query.

#### Writing to Column-Oriented Storage

LSM-trees are good for writing to column-oriented storage.  All writes first go
to an in-memory store, where they are added to a sorted structure and prepped
for writing to disk. It doesn't matter if the in-memory store is row or column
oriented.

> The above is essentially what Vertica does.

#### Aggregation: Data Cubes and Materialized Views.o

**Materialized aggregates**. Data warehouse queries often involve an aggregate 
function, such as `COUNT`, `SUM`, `AVG`, `MIN`, or `MAX` in SQL.  If the same
aggregates are used by many different queries, it can be wasteful to run the 
queries every time.

_Matierialized view_ is a way of creating a cache for these calculations and is
an acutal copy of the query results, written to disk.

A common special case of a materialized view is known as a _data cube_ or _OLAP cube_.
It is a grid of aggregates grouped by different dimensions. The advantage of a 
materialized data cube is that certain queries become very fast as they have 
effectively been precomputed.

A disadvantage is that a data cube doesn't have the same flexibility as querying
raw data.

## Encoding and Evolution

Applications change over time, when a data format/schema changes - correspoding changes
to application code often need to happen.

* Server-side applications may want to have _rolling upgrades_, deploying the new
version to a few nodes at a time
* Client-side applications are at the mercy of the user, who may not install the
update for some time.

This means old and new versions of code/data formats may coexist, compatibility
needs to be maintained in both directions

* **Backward compatibility**. Newer code can read data that was written by older
code.
* **Forward compatibility**. Older code can read data written by newer code.

### Formats for Encoding Data

Programs manage data in usually the following ways:

1. In memory, i.e. objects, structs, lists, arrays, etc.
2. Writing data to a file/sending over the network requires the data to be
encoded in a sequence of bytes, i.e. JSON.

Translation between the two above representations is called _encoding_ (aka 
_serialization_). The reverse is called _decoding_ (aka _deserialization_).

#### Language-Specific Formats

Many programming languages have built-in support for encoding in-memory objects
into byte sequences.  Java has `java.io.Serializable`, Python has `pickle` and
so on.

Convenience aside, these built-in encodings are tied to a specific language, making
it difficult to read in another language.  Also, it can pose a security risk as
the decode process needs to be able to instantiate arbitrary classes; if an attacker
gets the application to decode an arbitrary byte sequence, they can also 
instantiate arbitrary classes and remotely execute code.

##### JSON, XML, and Binary Variants

Standardized encodings can be written/read by many programming languages.

Despite flaws, such as ambiguity in number encoding, lack of binary string support,
and optional/lack of schema support, these three formats are good enough for many 
purposes.

###### Biary encoding

In organizations there's less pressure to use encoding like JSON, for example
you could use an encoding that is more compact/faster to parse.

JSON and XML use a lot of space compared to binary formats, leading to development
of JSON binary encodings like MessagePack, BSON, BJSON, etc. and XML variants like
WBXML and Fast Infoset.

##### Thrift and Protocol Buffers

Apache Thrift and Potocol Buffers (protobuf) are binary encoding libraries 
reliant on a schema for encoded data.o

Here's an example schema in the Thrift interface definition language (IDL):
```
struct Person {
    1: required string          userName,
    2: optional i64             farvoriteNumber,
    3: optional list<string>    interests
}
```

The integers are considered field tags and are referenced in the encoded
document to point back to the schema definnition, in other words - it is a compact
way of saying what field we're talking about.

Thrift also has two encoding formats, _CompactProtocol_ and _BinaryProtocol_.  
As the name implies, Compact protocol can encode the same data in a much smaller
size by packing the field type and tag number into a single byte.

Protocol Buffers are similar to the _CompactProtocol_ above, but does bit packing
a bit differently.

###### Field tags and schema evolution

Schemas inevitably change over time, how do Thrift and Protocol Buffers handle
schema changes while maintaining forward and backward compatibility?

In the context of these two encodings, If a field value is not set, it is omitted 
from the encoded record.  You can also change the name of a field in the schema,
since the encoded data only references the field tag.

You can add new fields to the schema, provided each new field gets a new tag 
number.  If old code reads data written by new code, including the new field it
doesn't recognize, it ignores the field.  This maintains forward compatibility;
old code can read records written by new code.

For backward compatibility, as long as each field has a unique tag, new code can
always read old data.  The only detail is if you create a new field, it cannot
be required because that check would fail if new code read data written by old code, 
because the old code will not have written the new field that you added.

Removing a field is similar to adding a field: only optional fields can be removed
and you can never use the same tag number again.

###### Datatypes and schema evolution

Changing datatypes of a field can also pose the risk of precision loss in types
like converting a 64-bit int to a 32-bit int.

##### Avro

Apache Avro is another binary encoding format that uses schema like previously mentioned.

Example:

```
record Person {
    string                  userName;
    union { null, long }    favoriteNumber = null;
    array<string>           interests;
}
```

One difference here is the lack of tag numbers, the encoding consists of values
concatenated together. To parse, you go through the fields in the order they
appear in the schema and use the schema to tell you the datatype.  It can only be
decoded correctly if the code reading the data is using the _exact same schema_
as the code that wrote the data.

###### The writer's schema and the reader's schema

Avro encodes the data using whatever schema version it knows about, that
schema may be compiled into the application.  This is known as the _writer's schema_.

When an application wants to decode some data, it expects the data to be in some schema,
which is known as the _reader's schema_.

The key idea with Avro is that the reader and writer schemas don't have to be the
same - only compatible.  When data is decoded(read), the Avro library resolves the 
differences by looking at the writer's schema and reader's schema side by side
and translating the data from the writer's schema into the reader's schema.

The two schemas are resolved by matching up fields by the field name.  If 
a field in the writer's schema isn't in the reader's schema, it is ignored. If
the code reading the data expects some field, but the writer's schema doesn't contain
a field of that name, it is filled in with a default value declared in the reader's
schema.

###### Schema evolution rules

With Avro, forward compatibility means you can have a new version of the schema
as a writer and an older version of the schema as reader.  Conversely, backward 
compatibility means you can have a new version of the schema as a reader and an
older version as writer.

To maintain this, you may only add/remove fields that have a default value

###### But what is the writer schema?

We can't just include the entire schema with every record as it negates the storage
savings from using Avro in the first place

Use cases for Avro:
* Large file with lots of records encoded with the same schema
* Database with individually written records, include a version number of the schema
version that was used.
* Sending records over network, two processes can negotioate schema version to use
for the connection lifetime.

###### Dynamically generated schemas

Avro can easily generate schemas from the relational schema and encode database contents
using that schema.

### Modes of Dataflow

This section covers some of the most common ways how data flows between processes
* via databases
* via service calls
* via asynchronous message passing

#### Dataflow Through Databases

Forward and backward compatibility is important in databases as it is common for
several different processes to access a database at the same time.  In conjunction 
with rolling upgrades on a service, some processes will run newer code while
some run older code.

To further explore this - a value in a database may be written by a newer version
of code while the same value is read by the older version of the code, so forward
compatibility is required.

Also, say a field is added to the record schema, and the new code writes a value
for that field.  And an older value reads the record, updates it, and writes back.
In this situation, the desirable outcome is that the old code leaves the new field intact
even though it couldn't be interpreted.

#### Dataflow Through Services: REST and RPC

Communication over a network is arranged commonly with two roles: _clients_ and
_servers_. The servers expose an API over the network, and the clients can connect
to the servers to make requests to the API. This API is known as a service.

Servers can be clients to another service, such as a database.  This approach
is commonly used to decompose large applications into smaller services by
area of functionality.  This way of building applications has been called a 
_service-oriented architecture_ (SOA), more recently known as _microservices 
architecture_.

When HTTP is used as the underlying protocol for talking to the service, it is
called a _web service_.

Two popular approaches to web services are REST and SOAP.

REST is not a protocol, rather a design philosiphy building upon the principles
of HTTP.  It emphasizes simple data formats, using URLS for identifying resources
and using HTTP features for cache control, authentication, and content type
negotiation.

SOAP is an XML-based protocol for making network API requests.  It comes with a
multitude of related standards and avoids using most HTTP features.

##### Current directions for RPC

The main focus of RPC is on request between services owned by the same organization,
typically within the same datacenter.  Whereas REST is the predominant style for
public APIs.

#### Message-Passing Dataflow

_Asynchronous message-passing systems_ are somewhere between RPC and databases.
They are similar to RPIC in that a client's request (message) is delivered to another 
process with low latency.  They're similar to databases in that the message is not sent via 
a direct network connection, but goes via an indermediary called a _message broker_,
which stores the message temporarily.

##### Message Brokers

In general, message brokers are used as follows: one process sends a message to a named
_queue_ or _topic_, and the broker ensures that the message is delivered to one 
or more consumers or subscribers to that queue or topic.  

A topic provides only one-way dataflow.  However, a consumer may itself publish 
messages to another topic, or to a reply queue that is consumed by the sender
of the original message.

##### Distributed actor frameworks

The _actor model_ is a programming model for concurrency in a single process.
Instead of dealing directly with threads, logic is encapsulated in _actors_. 
Each actor typically represents one client or entity, it may have some local state, 
and it communicates with other actors by sending and receiving async messages.

In _distributed actor frameworks_, this programming model is used to scale an application
across multiple nodes.  The same message-passing mechanism is used, no matter 
whether the sender/recipient is on the same or different nodes.

Location transparency works better in the actor model than in RPC, because the actor 
model assumes messages may be lost.

# Part 2: Distributed Data

There are many reasons as to why you would want to distribute a database:

* **Scalability** - if the data volume or read and write load grows larger than
what a single machine can handle, you can spread across multiple machines.
* **Fault tolerance/high availability** - Continued functionality of a service
even when one machine is down.
* **Latency** - Speed up reads and writes from users across the world by having
servers in various locations near your users.

### Scaling to Higher Load

The simplest approach is to buy a more powerful machine (_scaling up_) to 
scale to a higher load.  This is a _shared-memory architecture_.

The issue with this approach is that the cost to performance ratio is not
linear.

Another approach is the _shared-disk architecture_, which uses several
machines with independent CPUs and RAM, but stores data on an array of disks
that is shared between machines.  Commonly this is used for some data
warehousing workloads.

#### Shared-Nothing Architectures

_Shared-nothing architectures_ (i.e. _horizontal scaling_/_scaling out_) entail
that each machine running the DB software is called a _node_. Each node uses
its CPUs, RAM, and disks independently.  Coordination between nodse is done on
the software level.

Shared-nothing architectures require the most caution from the engineer as
there are constraints and trade offs.

#### Replication Versus Partitioning

There are two common ways data is distributed across multiple nodes:

* **Replication** - Keeping a copy of the same data on several different nodes,
potentially in different locations. Replication provides redundancy: if some
nodes are unavailable, the data can still be served from the remaining nodes.
Replication can also improve performance.

* **Partitioning** - Splitting a big database into smaller subsets so that
different partitions can be assigned to different nodes (i.e. _sharding_).

## Replication

_Replication_ means keeping a copy of the same data on multiple machines that 
are connected via a network.

Replication is easy if your data doesn't change over time, you just need to
copy the data to every node once.  The difficulty of replication lies in 
handling changes to replicated data.

There are three popular algorithms for replicating changes between nodes:

* **single-leader**
* **multi-leader**
* **leaderless**

There are many trade-offs to ocnsider with replication: for example, whether to
use synchronous or asynchronous replication, and how to handle failed replicas.

### Leaders and Followers

The most common solution for database writes being processed by every replica
is _leader-based replication_, which works as follows:

1. One of the replicas is the _leader_.  Writes must be sent to the leader, 
which first writes the new data to its local storage.

2. The other replicas (_followers_) receives data from the leader when a new
write passes through the leader and updates its local copy of the database
accordingly, by applying the writes in the same order they were processed
on the leader.

3. When a client wants to read from the database, it can query either the leader
or any of the followers. Writes are only accepted on the leader.

This mode of replication is built-in to many RDBMSs, like PostgreSQL (>=v9.0),
MySQL, Oracle Data Guard, and SQL Server's AlwaysOn Availability Groups. Some
NoSQL databases that have this include MongoDB, RethinkDB, and Espresso.
Distributed messages brokers use it as well: Kafka and RabbitMQ highly available
queues.

#### Synchronous Versus Asynchronous Replication

An important detail of a replicated system is whether the replication 
happens _synchronously_ or _asynchronously_.  

Normally replication is quite fast, but there's no guarentee of how long it
might take.  There are times where followers might fall behind a leader by
several minutes (a follower is recovering from a failure , network issues, 
etc.).

Sychronous replication guarentees that the follower will have an up-to-date
copy of the data consistent with the leader.  The downside is that if the 
follower doesn't respond (network crash, nuclear explosion), the write 
cannot be processed.  The leader blocks all writes until the replica is 
available again.

All followers being synchronous is impractical - usually enabling synchronous
replication means _one_ of the followers is synchronous and the others are 
async. If the sync follower fails, one of the async nodes is made sync,
guarenteeing that you have two up-to-date copies of the data on at least
two nodes (aka _semi-synchronous).

Often, leader-based replication is completely asynchronous, so if the leader
fails and cannot recover, any writes not replicated are lost - meaning that
a write is not guarenteed to be durable.

Research on Replication
> It can be a serious problem for asynchronously replicated systems to lose
> data if the leader fails, so researches have continued investigating
> replication methods that do not lose data bust still provide good performance
> and availability.
>
> _Chain replication_ is a variant of synchronous replication that has been 
> successfully implemented in a few systems such as Microsoft Azure Storage.

#### Setting Up New Followers

The process for setting up new followers looks like this:

1. Take a consistent snapshot of the leader's database
2. Copy snapshot to the new follower node
3. Follower connects to the leader and requests all data changes that occurred
since the snapshot was taken.  The exact position in the leader's replication
log is required. It's called the _log sequence number_ in PostgreSQL, and the
_binlog coordinates_ in MySQL.
4. When the follower processed the backlog of data changes since the snapshot,
it has _caught up_ and can process changes from the leader as they happen.

#### Handling Node Outages

The goal is to keep the system as a whole running despite individual node 
failures, and keep the impace of node outages small.

##### Follower failure: Catch-up recovery

On local disk each follower keeps a log of data changes from the leader.  If it
crashes and restarts, it uses this to recover to its last known transaction 
before the fault occurred.  Then call the leader and request all changes since
the outage.

##### Leader failure: Failover

In this scenario, one of the followers needs to be promoted to leader and 
can be done for manually or automatically.

Automatic failover process:

1. _Determining that the leader failed_, one option is to set a timeout, if 
a node doesn't respond in a set time interval, its assumed dead
2. _Choosing a new leader_, election process
3. _Reconfiguring the system to use the new leader_, clients now need to send
their writes to the new leader

Possible failover problems:

* New leader may not have received all writes from the old leader before it
failed, common solution if the old leader comes back is for old leader's 
unreplicated writes be discarded
* Discarding writes is dangerous if other storage systems outside of the db
need to be coordingated wiwth the database contents.
* Two nodes may both think they're the leader, there are mechanisms to shut
down one node if two leaders are detected.
* Determining the timeout when a leader is declared dead

#### Implementation of Replication Logs

How does leader-based replication work under the hood? Several different
replication methods are used in practice.

##### Statement-based replication



### Problems with Replication Lag
### Multi-Leader Replication
### Leaderless Replication

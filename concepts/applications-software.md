[Home](./../README.md) / Concepts / Applications Software

# Applications Software
 
- [Scope](#scope)
  - [What it is](#what-it-is)
  - [What it is not](#what-it-is-not)
- [Observations](#observations)
- [Principles](#principles) 
  - [Ontology](#ontology-first)
  - [Storage Design](#independent-storage-design)
  - [Runtime Modelling](#independent-runtime-modelling)

## Scope

The patterns described here represent an approach to building **data-focused enterprise applications software**. Historically this has sometimes been called "Line of Business" software.

### What it is

This type of software has the following characteristics:

 - The information it stores is both **long-lived and valuable**.
    - Ensuring the *validity* of the data is extremely important 
    - If the software ceased working but the data was unaffected, it would represent an interruption of service (possibly a loss of business) but not a loss of property
    - If the data were lost but the software was unaffected, it would be a catastrophic and unrecoverable loss. 
    - The monetary value of the data itself is at least in the millions of dollars. Corruption or loss of the data would likely be a [material event](https://en.wikipedia.org/wiki/Materiality_(auditing)) for the organization

 - The information describes **human activities**
    - Even when the content concerns digital or mechanical things or processes, the context is humans working with those machines and materials.
    - The information exists, the human activities which produce or modify the information are as the are. The software and its human designers and maintainers have limited ability to oppose or proscribe concepts due to conceptual difficulty or mechanical inconvenience. Instead, their role is to effectively model and facilitate the information and processes as they actually are.
    - Human activities constantly change, and so must the data stores and software which encode and process them. The software layer in particular must be built in a way that it can change substantially over time. It must *enable* the human activities both to function in their current form and to change over time, not be a *constraint* preventing change.
    
 - The users are a relatively small group who *must* use the software to execute a practical task

### What it is not

This kind of software is fundamentally different from some other familiar kinds of software. It is particularly **not** the following:

 - **Systems software**
   - Exists to standardize and optimize digital mechanical concerns. Whether that is a POSIX-compliant operating system, an HTTP server, or an ECMAScript interpreter, its purpose is to model mechanical processes (memory allocation and file IO, a request protocol, a language spec, etc.), not changing and unique human activities. 
   - Often has no data content at all, or is limited to low-level mechanical content that reflects machine rather than human activity concepts (file system, cookie cache, serialized AST, etc.)
   - Value is in permanence and standardization of the procedural action and API, sometimes on a global scale over decades of time.  
 - **Games**
   - Like business applications, games often do have significant amounts of data and even may model human activities. Unlike business applications, the role of the game designer is to describe and even impose an environment for a prospective audience, not model a process for an existing group of humans performing some practical activity. 
   - The primary concern of a game is to deliver an *experience* that a user will pay for, whether with money or with time and attention.
   - Users can choose to use the software are not.
 - **Digital Media**
   - Interactive media -- including media that allows purchase such as an online store -- are more similar to games in that they ultimately deliver an *experience*, whether that is an experience of consuming and commenting on interesting information, being entertained, or exploring and discovering things to buy. Such web sites and web apps may integrate with a business applications back end for processes such as billing and fulfillment.
   - Once again, users can choose to use the software or not.

## Observations

Over combined decades of building, using, supporting and retiring data-focused enterprise applications software, the following patterns became clear: 

### Data is primary
 - **Data is long-lived, often on the order of decades.** Data may *migrate* from one data store to another; attributes may be added or reorganized; but the content itself is the thing of value for the business. In the most serious cases, a record has contractual, physical, or financial significance. 
   - *Errors represent actual legal or financial risks or losses.*

### Applications change and fail
 - **Applications have bugs**.  
   - *The software layer cannot be responsible for guaranteeing data **validity**, because it **will** fail.*
 - **Applications change**, and more frequently than data stores. New technologies emerge, old skills are lost, trends come and go, and business goals change. 
   - *The application software should be able to change significantly without changing the data store*
 - **Frameworks change** in API, norms, and architecture even more often than business applications. 
   - *Wedding important semantics OR mechanics to a framework guarantees that, given enough time, **no one** will understand the code.*

### Applications come to define business processes
 - **Applications encode business logic in source code**. In practice, the existence and composition of source code becomes *the* source of truth for how a business process actually happens. This is not desirable; it is reality. 
   - *Business logic must be clear and distinct from implementation mechanics*
 
### Human thought and skill cannot be automated

*In general*

 - Building things requires thought and skill
 - Only humans have thought and skill
 - It is far simpler (and within human capability) to build *things* than it is to build *thing-making things*. Many people can do wood building [framing](https://en.wikipedia.org/wiki/Framing_(construction)). No machine can replicate the "simple" thought and skill applied by humans to cut and nail 2x4s in the right manner and in the right places. If such a machine existed, it would necessarily be very complicated and hard to build or maintain. The same is true for all the trades, including digital machine building -- otherwise known as software engineering.
 - Building a thing is not the same as inventing a thing. People who do framing are not "reinventing framing". They are executing a well known process using human thought and skill. 

*When building software applications*

 - The following concerns represents distinct *and differing* requirements, and thus require different thoughts and skills to design and implement:
   - **Data storage**: selection, design and implementation of data stores
   - **Data representation at run time**  including the choice of language and applicable idioms for representing records 
   - **Procedural logic** for creating and modifying data, whether through basic CRUD actions or more complex business logic
   - **Data serialization** for arms-length APIs

 - These concerns should be addressed independently by applying patterns that do not require the implementation in one domain to know much about the implementation of the others. 

## Principles

### Ontology First

The **ontology** is the first and most real aspect of the system. 

The ontology is the shape of the information itself -- which in most business systems in the *actual value* of the system. The work of ontology involves identifying what concepts need to be recorded, how they relate to each other, and what static rules can be asserted about data *validity*.  

An ontology is not a database schema. The concept of an invoice and its line items exists the same whether they are stored in a flat file, a relational database, a document store, or a clay tablet. 

### Independent Storage Design

**Storage design** involves selecting and specifying how to store (persist) an ontology in digital form. It requires a broad understanding of storage types, including at a minimum relational databases. 

Decisions include where and how to *normalize* (or not), *index* (or not), *constrain* (or not), *compress* (or not), *partition* (or not), etc. 

The **data itself** should be valid, accessible, and understandable **without the application layer**.  As noted above, *applications change and have bugs*. It must be possible for specialists and support personnel to interact directly with the data store itself, and in a way that the data can be understood.

#### Practical implications

Most importantly:

> Storage design requires human thought and skill. It cannot be automated. 

Storage design and implementation -- often realized in the practical task of writing SQL DDL statements for a specific brand of relational database -- is one of the framing tasks of data applications engineering. In other words, *don't try to make a storage definition-generating machine*. It will be far more complex than using a human who knows what they are doing, and will never match the quality of human work. 

At the most, limit generated DDL to the simplest and most obvious cases, while allowing human work to seamlessly blend with the generated DDL.

**Relational databases are almost always the right choice** for most application data, both because they offer mathematically-proven validity and consistency if implemented correctly, and because they provide well-understood and powerful access to the data through SQL.

**Splitting data between multiple data stores is a very expensive decision.** It increases complexity, and is a barrier to understanding the data without the software layer. As such it demands dramatic benefits to justify. 

Examples where this may be justified depending on the scale, costs, and performance demands include:
 - Storing file blob content in a separate service, with only metadata in a database 
 - Replicating and denormalizing some data into a search database (e.g. elasticsearch, solr)
 - Storing raw JSON inputs to an API in a document database, with a limited subset imported to a main relational database
     
### Independent runtime modelling

**Runtime model design** involves deciding how to encode the [ontology](#ontology-first) in a specific programming language. It requires understanding the semantics and mechanics of a specific language, and using those to represent the ontology faithfully, flexibly, and in a way that works well with the known patterns of the how the data will actually be used. This can only be known by a human.

Decisions include when or how to *build and represent collections*, *communicate which storage actions are valid*, *lazy load* (or not), *mutate* (or not), *support batch operations* (or not), and *allow free-form searching* (or not).  

#### Practical implications 

> Runtime model design requires human thought and skill. It cannot be automated. 

Runtime model design is another of the framing tasks of data applications engineering. In other words, *don't try to make a runtime model-generating machine*. It will be far more complex than using a human who knows what they are doing, and will never match the quality of human work. 

At most, limit generated models to the simplest and most obvious cases, while allowing human work to seamlessly blend with the generated code. 

> Runtime models do not know about storage implementation

Runtime models encode the ontology, **NOT** the data store schema. Runtime models represents information itself, **NOT** database rows, or flat file lines, or clay tablet chunks. 

## WIP
More to come.

##### Attribution

Author: Joshua Honig. Copyright 2019 Mallowfields LLC.
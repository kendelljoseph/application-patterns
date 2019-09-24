# Adapter Pattern

The Adapter pattern is a language-independent set of concepts and techniques for implementing the data layer of an application. It was developed over combined decades of experiments, debates, and production implementations by an enterprise application development team at Steelcase, of which I was a member. Yes, normal human software engineers at a sleepy old office furniture manufacturer in West Michigan.

This document covers the what and how of the pattern. The foundation of the whys are discussed at length in [Applications Software](../concepts/applications-software.md).

## Contents
- [Concepts](#concepts)
  - [Expressing intent](#expressing-what-is-intended)
  - [Object Graphs](#no-unstored-object-graphs)
  - [Collection Synchronization](#no-automatic-collection-synchronization)
- [API](#adapter-aPI)
  - Model
  - Connection
  - Adapter
- Implementation

# Concepts

## Expressing what is intended

One of the primary goals of the Adapter pattern is to allow the authors of the data access layer to clearly express which actions are valid and intended. 

This is very different from many ORM models, or non-relational equivalents, which simply transcribe SQL or other data store protocol semantics to the runtime language. In other words, ORMs let you do most or all actions on most or all models; they just provide mechanics to allow one to execute SQL-like actions while spelling in a runtime language. Thus they support complex dynamic queries, and generate a standard set of methods for most model types corresponding to all the CRUD actions. 

In contrast, the Adapter pattern exposes only and exactly the data and functionality that is intended. Intention is a human idea, and must be expressed by one human (an author of the data layer) to another (a user or reviewer of the data layer--which might be the same human a few days or years later).
 
## No unstored object graphs

This point is not required by the adapter semantics itself, but is a key technique for simplifying the mechanics of the data layer -- and making things easier for humans to understand and maintain. 

Building object graphs before they are stored is a major source of complexity it data layers, whether hand-rolled or through framework ORMs. I have built such mechanics myself more than once. It was a great puzzle and I felt very proud of my clever work, and I suspect many ORM authors and users also thrill at the delightful algorithms used to make sure everything is saved in the right order, or possibly rolled back if something goes wrong part way through.

It is not worth it. Period. Do not do it. Instead, provide `Create` actions that 
 - Express exactly what inputs are required to store a *valid* record of the applicable type.
 - Require valid models (not model IDs) for required related models
 - Only return a model if the storage action (i.e. `INSERT` in SQL) succeeds 

This implies something else, too:

> In languages that support it, model constructors should be non-public

You don't ever *want* a runtime instance that doesn't correspond to a record that is or at least was in the data store. One way to **communicate this intention** is by making the constructors on model classes non-public, and instead only providing model instances through the corresponding adapter's `Create` or `Get/Find` methods. 

## No automatic collection synchronization

Like fancy model graph saving, collection synchronization is a major source of complexity in ORM frameworks and confusion and woe for developers. 



# Adapter API

The basic API is a set of concepts and shapes that serve to 
- Clearly express what data storage actions are valid
- Organize storage APIs in human-scale, discoverable groups
- Insulate the runtime data model from storage implementation details
- Facilitate clear mechanics throughout the storage logic
 
These concepts and shapes are language-independent, and can in fact be implemented with very similar semantics in very different languages.
  
## Model

A `model` represents the data for a single instance of a single concept at runtime. Unlike many contemporary data access patterns, 

> Models **do not know** how they are stored or serialized

Models contain only properties, fields, and methods related to their runtime data concepts (which derives from [ontology, not a database schema](../concepts/applications-software.md#ontology-first)). 

Models may require an injected **connection** so that they can provide semantically useful APIs for retrieving related objects, but as noted below the **connection** does not expose any information about the storage mechanics.
 
The model does not change at all when:

 - Backing storage changes from a relational database to a document database
 - A new serialization format must be supported (e.g. JSON, XML, protobuf)

### Intention at the model level

#### *What things can be changed?*

Many models have some attributes that are intended to be mutable and others that are immutable. Mutable attributes can be updated one or many times after the record is created, while immutable attributes are provided exactly once when the record is created, or even generated by the data store, and then should never be changed. "Should" is another human concept can only be known and expressed by a human.

When the language supports it, immutable attributes should be clearly expressed in the language-specific idiom:

C#
```c#
public class User {
  ID        int    { get; }  // Immutable: read-only property
  UserName  string { get; }  // Immutable: read-only property
  GivenName string { get; set; } 
}
```

JavaScript - function class with underscore convention
```javascript
function User(id, userName) {
  this._id = id
  this._userName = userName
}

User.prototype.givenName = ""

Object.defineProperties(User.prototype, {
  id:       { get: function() { return this._id }},
  userName: { get: function() { return this._userName }}
})
```

JavaScript - class with closure
```javascript
class User {
  constructor(id, userName) {
    Object.defineProperties(this, {
      id:       { get: function() { return id }},
      userName: { get: function() { return userName }}
    })
    this.givenName = ""
  } 
}
```

Go
```go
type User struct {
  id        int
  userName  string
  GivenName string
}

func (u *User) ID() int {
  return u.id
}

func (u *User) UserName() string {
  return u.userName
}
```

#### *What things are related?*

If I have an `Order`, what other models are directly related and reasonable to retrieve? ORM frameworks often express these through "relations", "navigation properties", or similar concepts. The point here is to include on the model only those related models or collections that are conceptually directly related. 

For example, the one *required* `Customer`, one *optional* `SalesPerson`, and zero to many `LineItem`s associated with an `Order` model are fundamentally part of the order concept itself. They should be present on / accessible directly from the model. Likewise, the one `Order` associated with a `LineItem` is part of the `LineItem` concept itself, and so should be accessible directly from the `LineItem`.

In contrast, the zero to many `Orders` for a `Customer` are **not** fundamentally part of the customer concept itself. The `OrderAdapter` (see [model adapter](#model-adapter) below) should provide a `FindAll` method for finding the `Orders` for a `Customer`, but the `Customer` model should not include an `Orders` collection.

## Model Adapter

**Each** model type has a corresponding `Adapter` type which defines only and exactly the valid storage actions for that model type, and in an implementation-independent manner. When the language supports it, the model adapter type should be an [interface or abstract class](https://en.wikipedia.org/wiki/Protocol_(object-oriented_programming)).

The standard verbs are:

|Verb|Description|
|-|-|
|`Get`|Retrieve one or zero models given parameters which will match one or zero models|
|`GetAll`|Retrieve zero to many models given parameters which correspond to a clearly defined set|
|`Find`|Retrieve one (the first) or zero models given predefined or dynamic parameters which may or may nor match zero, one, or many models|
|`FindAll`|Retrieve zero to many models given predefined or dynamic parameters which may or may nor match zero, one, or many models|
|`Create`*|Accept all inputs required to store a new valid record, store that record, and return the model|
|`Save`**|Update the storage of a model|
|`Delete`**|Remove a single model from storage|

### \* Non-Public Create

Depending on the preferences of the team, the ability of the language to express it, and the specific object model, the `Create` method for a particular model may be non-public, and instead exposed via a factory method on a related adapter or model. Both of the following are perfectly dandy. Just pick one and stick with it:

C# : `Create` on `LineItem`'s own **adapter**
```C# 
var con   = GetConnection()
var order = con.Order.Get(oderId)
var line  = con.LineItem.Create(order, product)
```

C# : `Create` indirect through factory method on `Order` **model**
```C#
var con   = GetConnection()
var order = con.Order.Get(oderId)
var line  = order.AddLine(product) // Calls internal con.LineItem.Create()
```

### \*\* Non-Public Delete, Save

Depending on the preferences of the team and the ability of the language to express it, the `Delete` and `Save` methods may be hidden from the public API and instead exposed as instance methods on the corresponding model class. Both of the following are perfectly dandy. Just pick one and stick with it:

C# - Update methods on Adapter
```C#
var con     = GetConnection()
var order   = con.Order.Get(oderId)
order.State = OrderState.Complete
con.Save(order)
```

C# - Update methods on model
```C#
var con     = GetConnection()
var order   = con.Order.Get(oderId)
order.State = OrderState.Complete
order.Save()  // Calls internal method con.Order.Save()
```

### Neutral Semantics

A model adapter uses semantics that do not imply a specific kind of data store. They do not change if storage switches from a direct connection to relational database to a protobuf API over gRPC. So we `Get` (neutral) instead of `Select` (SQL), and `Save` (nuetral) instead of `Post` (HTTP). 

If you like `Retrieve` better than `Get`, or `Remove` better than `Delete`, or `Update` better than `Save`, or `Search` better than `Find` as implementation-neutral words for these ideas then go for it. Just be consistent.

### Singular and Plural Semantics
In the table above I used for example `Get` and `GetAll`. If you want to use `GetOne` and `GetList`, or `Get` and `GetMany` then thumbs up, just be consistent. 

### Overload Semantics
A single adapter may have multiple `Get`- and `Find`-type methods, corresponding to different well-defined inputs from which you might get or find one or more models. In languages that support overloading it makes sense to use the overload technique here instead of different method names. In dynamic languages either can be implemented, and it is a matter of taste which is "clearer". In other languages overloading is not possible at all and so different names must be used. Once again, pick a convention and stick to it.

C# - Overloads
```C#
public interface IOrderAdapter {
  Order Get(id long)      // Get by primary key id
  Order Get(code string)  // Get by the order 'number' that customers see 
}
```

JavaScript - Overloads
```javascript
OrderAdapter.prototype.Get = function(arg1) {
  if ("number" === typeof arg1) {
    return this._getById(arg1)
  } else if ("string" === typeof arg1) {
    return this._getByCode(arg1)
  } else {
    ...
  }
}
```

JavaScript - Separate Methods
```javascript
OrderAdapter.prototype.GetByID   = function(id)   { /*...*/ }
OrderAdapter.prototype.GetByCode = function(code) { /*...*/ }
```

Go - Separate Methods
```go
type OrderAdapter interface {
  GetByID(id int)        (*Order, error)
  GetByCode(code string) (*Order, error)
}
```

### Only Include Required Actions

Some models, like log entries, are intended to be completely immutable. They should neither be updated nor deleted after they are created, except perhaps by back end rotation mechanism implemented outside the application layer. In these cases, don't include a `Save` or `Delete` method on that model's adapter. 

User models are often updatable and can be permanently disabled but can't actually be deleted because it would destroy historical records; so don't include a `Delete` method for users.

In actual applications, most models are not normally *searched for*, and so should not have any `Find` or `FindAll` methods.

The generalization here should be clear: The point is to express and expose only the intended, reasonable, valid storage actions for each model.

## Connection

The root interaction with data storage is through a `connection` object. The connection object's public API does not reveal what kind of data store (or even multiple stores) that it connects to. When the language supports it, this should be an [interface or abstract class](https://en.wikipedia.org/wiki/Protocol_(object-oriented_programming)).

The main purpose of the connection is to be a gathering point for the individual model adapters. For a small application they may all be at the top level:

*Go*
```go
type Connection interface {
  User      UserAdapter
  Order     OrderAdapter
  LineItem  LineItemAdapter
  Customer  CustomerAdapter
}

// Used like:
con.Order.GetByID(orderId)
```

For a larger application they may be grouped by concept, which may or may not correspond to an actual storage group or schema:

*C#*
```C#
public interface IConnection {
  Security ISecurityAdapters { get; }
  Sales    ISalesAdapters    { get; }
}

public interface ISecurityAdapters {
  User     IUserAdapter { get; }
  Role     IRoleAdapter { get; }
}

public interface ISalesAdapters {
  Order    IOrderAdapter    { get; }
  LineItem ILineItemAdapter { get; }
}

// Used like:
con.Security.User.Get(username)
```

### Dot Discoverability

Code must be understood by humans. It is communication from one human to another in a form that also encodes actual mechanics that a computer can execute. As described at length above, the adapter pattern aims to aid understanding by making the number of options in any given situation both explicit and limited. For example list of "storage actions for a user" is explicit on the user model adapter, and limited to what is reasonable, relevant, and intended.

The particular shape of the APIs described above is in part influenced by how we actually write code -- in editors and IDEs with moderate intelligence, which support auto-completion and inline documentation. In the vast majority of contemporary general-purpose languages, the period or dot (`.`) is the member accessor, and also the trigger for the editor to display a list of *things that can come next*. 

The particular semantics of `connection[.group].model.action` directly aids this by grouping the lists of what comes next into small steps, where the options at each step are limited and relevant. With two or three branch levels starting from `connection`, we can express even very large object models succinctly. 

In practice, this means a consuming developer (including oneself after enough time has elapsed to forget details), can discover a substantial amount about the modelled domain just by typing a few lines in a capable code editor.


# Implementation

To state the obvious, storage actually has to be implemented in some specific place or places. Regardless of the format and protocol, this requires code to bind to the specific implementation details and mechanics of the storage.

## Storage and model as separate decisions

As discussed at length in [Applications Software](../concepts/applications-software.md#principles), the work of designing a runtime data model and the work of designing appropriate storage for that model are separate concerns. When implemented well, they almost always have subtle differences. 

Put visually, the graph of conceptual references looks like this:
 
```
    [Ontology]
      /   \
     /     \
[Model]  [Storage]
```

And NOT like this:

```
 [Model]
    |
[Storage]
```

or this:

```
[Storage]
    |
 [Model]
```

What remains is the task of mapping the model to the storage. This is necessarily also human work, though portions of it can be automated. Regardless of whether it is built by a person or machine, the point is that what needs to be built is this:

```
[Model] [Storage]
   ↓        ↓
  [*Mapping*]
```

NOT this:

```
 [Model]  
    ↓
[*Storage*]
```

or this:

```
[Storage]  
    ↓
[*Model*]
```

## Storage Adapter
Within the adapter pattern, this mapping code is implemented in the storage adapter(s) for each model type. Its role is to implement the platform-neutral API of the [model adapter](#model-adapter) through platform-specific interactions with the actual storage provider.

## Mechanics, Mapping, and Code Generation

### Principles

Clarity through 
 - Short call stacks
 - Hard coded SQL
 - Hard coded mappings
 
### Mechanics
For any given storage type and platform and runtime language certain *mechanics* of the storage adapters for a system will be the same. Regardless of the actual model shape or the specific parameter values required to insert a new record, all SQL storage adapters need to

 - Obtain a connection from the connection pool
 - Create parameters derived from a provided model instance
 - Execute a SQL INSERT statement
 - Possibly retrieve the output of the INSERT statement if the platform supports it, OR execute a SELECT query, in order to retrieve values generated by the database and assign them to the model. The most frequent case is a generated primary key
 - Release the connection back to the pool even if something goes wrong.

Similar mechanics are required for `UPDATE`, `DELETE`, and `SELECT` 
operations. 

For each model type, we need:

- Literal SQL statements
- Code to read specific attributes into database parameters
- Code to 
    - Read values from a database row
    - Convert the values to the correct runtime type
    - Assign the converted values to the correct model attributes
 
### Mapping
One of the basic tasks in this realm, as in other protocol serialization and deserialization tasks, is mapping the names of runtime attributes to the names of tables and columns in a database. 

Humans are good at thinking, but back at mechanical tasks. Machines are good at mechanical tasks, but can't think at all. And that is the irreducible challenge of tasks like this. 

There are several options:

#### All by hand

The advantage here is the simplicity of tooling. There is no thing-making thing to configure and manage. 

The other very important advantage is in clarity of the actual code. Without complex ORMs handling storage operations, the call stack from business logic to storage should be short and clear. The SQL should be present in the source code, as are the procedural mapping statements. When something goes wrong, we can see exactly what is happening.

##### Dumb bugs

The downside is not time but *mistakes*. Compared to all the other work involved in building an enterprise data application, writing rote SQL statements by hand is not time-consuming. 

Rather, compared to all the other work involved in building an enterprise data application, writing rote SQL statements and mapping statements by hand is one of the easiest ways to introduce bugs. Latent, gnarly, hard-to-find bugs.

When a bug is identified, the clear mechanics of hand-rolled code often help debugging to be a simple exercise. Unfortunately, mapping mistakes easily lead to bugs that are not noticed, or whose effects do not obviously point to a mapping problem.

A partial workaround is to write exhaustive unit tests for all basic CRUD operations...but the same mistakes that can be made in mapping code can be made in the code that tests the mappings. Further, the mapping tests are often little more that repeats of the *implementation* of the mapping. 

##### Ambiguity

Another downside is ambiguity of intent, which is exactly the kind of thing we are trying to avoid. Ontologies, models, and storage implementations all include *human decisions*, such as about which attributes should be immutable. Implementing the mapping and SQL code requires being aware of and implementing these decisions. The problem is that, unless everything is meticulously commented, a reader of the code does not know whether the choices actually implemented are intentional, even when the mappings are correct. 

- *Is it intentional that X column in the database is never loaded at runtime?*
- *Is it intentional that attribute Y is omitted from the UPDATE statement?*
- *Is it intentional that a column is nullable in the database but not on the model?*

#### Use an ORM

The purported advantage here is speed and reduction in mistakes. Whether through convention or configuration, SQL code and/or runtime model code is generate from one gold source (often the mirror SQL or runtime model, possibly with embedded metadata). So in general it reduces the number of places where mapping errors occur to a single configuration point.

The downsides are many. Most ORMs do not simply automate mapping, they instead interject themselves throughout all storage code, and even into the model and/or storage layer. For all the reasons described here and in [Applications Software](../concepts/applications-software.md), this is bad, frustrating, and unnecessary. ORMs are the poster child of thing-making things, that introduce immense complexity, and yet cannot match the quality of a human doing the same "simple" tasks. 

Another challenge with ORMs is that most generate behavior and SQL code at runtime rather than compile time. This makes debugging difficult, because the actual SQL statement exists nowhere in source code, and the actual mapping statements are dynamically organized and executed. [Active Record](https://guides.rubyonrails.org/active_record_basics.html) provides an extreme example, where basic CRUD operations tunnel through dozens of layers of call backs and evaluated code generated at runtime. 

#### Code generation by hand

As you could likely anticipate, this is the middle way that I advocate. Done well, this approach hews to the following principles:

- **There is no *runtime* dependency.** We are generating code, not behavior. The code does not know or care how it came to exist, and the potential complexity and problems with the code-generating thing is at least confined to authoring time. 

- **The code looks like a human wrote it**. 

- **Generated code can be mixed with hand-written code**

- **Intent is explicit**


#### Automated *validation*

A related approach automates analysis of code rather than generation of code. Moderately simple parsers and linters can be built to validate sanity checks like the following:

 - An attribute is not mapped to more than one parameter
 - A column is not mapped to more than one attribute
 - The table name in the SQL is correct

For full validation, there is no avoiding that human intent must be recorded somewhere, whether in a separate data file or in embedded metadata.

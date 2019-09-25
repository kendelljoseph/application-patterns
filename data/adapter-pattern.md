[Home](./../README.md) / Data / Adapter Pattern

# Adapter Pattern

The adapter pattern is a language-independent set of concepts and techniques for implementing the data layer of an application. It was developed over combined decades of experiments, debates, and production implementations by an enterprise application development team at Steelcase, of which I was a member. Yes, normal human software engineers at a sleepy old office furniture manufacturer in West Michigan.

Additional material on the motivations and philosophy of this pattern are discussed at length in [Applications Software](../concepts/applications-software.md).

## Contents
- [By Example](#by-example)
- [API](#adapter-api)
  - [Model](#model)
  - [Model Adapter](#model-adapter)
  - [Connection](#connection)
- [Implementation](#implementation)
  - [Storage Adapter](#storage-adapter)
- [Principles](#principles)
  - [Clarity of intent](#clarity-of-intent)
  - [Clarity of mechanics](#clarity-of-mechanics)
  - [Achieving clarity](#achieving-clarity)
  - [Independence of model and storage](#independence-of-model-and-storage)
- [Design Points](#design-points)
  - [Unstored Object Graphs](#no-unstored-object-graphs)
  - [Collection Synchronization](#no-automatic-collection-synchronization)
  - [Graph Upates](#no-automatic-object-graph-updates)
  - [Reference Equality](#no-automatic-reference-equality)

# By Example
<details><summary><em>Expand for example</em></summary>

The following examples are in C#, but as later examples illustrate this is a language-independent pattern. 

1. Define a **model**, which knows its attributes and immediate relations, but not how it is stored or serialized:

    ```c#
    public class Order {
      private IConnection connection;

      // Constructor is internal
      internal Order(IConnection connection)
      {
        this.connection = connection;
      }

      // Public attributes
      public int ID { get; internal set; }
      public string Code { get; internal set; }
      public int CustomerID { get; internal set; }
      public string Comment { get; set; }

      // Relations
      internal Customer _customer;
      public Customer Customer
      {
        get
        {
          if (_customer == null) _customer = connection.Customer.Get(CustomerID); 
          return _customer;
        }
      }

      internal List<OrderLine> _lines;
      internal IReadOnlyCollection<OrderLine> _linesRead;
      public IReadOnlyCollection<OrderLine> Lines
      {
        get
        {
          if (_lines == null) _lines = connection.OrderLine.GetAll(this).ToList(); 
          if (_linesRead == null) _linesRead = _lines?.AsReadOnly(); 
          return _linesRead;
        }
      }
    }
    ```

2. Define a **model adapter**, which expresses the valid storage operations for the model in platform-neutral semantics, but does not implement those operations:

    ```c#
    public interface IOrderAdapter {
      Order Create(Customer customer);
      Order Get(int id);
      Order Get(string code);
      IEnumerable<Order> GetAll(Customer customer);
      void Save(Order order);
      void Delete(Order order);
    }
    ```

3. Define a **connection** which collects the model adapters:

    ```c#
    public interface IConnection {
      IOrderAdapter    Order     { get; }
      ICustomerAdapter Customer  { get; }
      IOrderLine       OrderLine { get; }
    }
    ```

4. Create a **storage adapter** which implements the **model adapter** through concrete interactions with some underlying data store

    ```c#
    // Shared mechanics
    internal abstract class SQLAdapterBase<T> {

      protected readonly IConnection connection;
      internal SQLAdapterBase(IConnection connection) => this.connection = connection;
      
      protected abstract string SQLInsert { get; }
      protected abstract Dictionary<string, object> GetInsertParams(T model);
      protected abstract T Construct();
      protected abstract void ReadRow(DbDataReader reader, T model);

      protected void Insert(T model)
      {
        var dbparams = this.GetInsertParams(model);
        using (var sqlCon = GetSqlConnection())
        {
          sqlCon.Open();
          var cmd = sqlCon.CreateCommand();
          cmd.CommandText = this.SQLInsert;
          foreach (var pair in dbparams)
          {
            cmd.Parameters.AddWithValue(pair.Key, pair.Value);
          }
          using (var reader = cmd.ExecuteReader())
          {
            if (!reader.Read())
              throw new InvalidOperationException("Insert statement did not return any rows");
            this.ReadRow(reader, model);
          }
        }
      }

      /* ... etc ... */
    }

    // Model-specific mechanics
    internal class OrderSQLAdapter : SQLAdapterBase<Order>, IOrderAdapter 
    {
      internal OrderSQLAdapter(IConnection connection) : base(connection) { }

      override protected string SQLInsert => @"
        INSERT INTO dbo.Order (
          CustomerID
        )
        OUTPUT inserted.*
        VALUES (
          @CustomerID
        )
      ";

      override protected Dictionary<string, object> GetInsertParams(Order model) {
        return new Dictionary<string, object> {
          { "CustomerID", model.CustomerID }
        };
      }

      public Order Create(Customer customer) {
        var order = new Order(this.connection) {
          CustomerID = customer.ID,
          _customer  = customer
        };

        // Order ID and Code are generated by the database
        base.Insert(order);
        return order;
      }
       
      protected override void ReadRow(DbDataReader reader, Order model)
      {
        model.ID = Convert.ToInt32(reader["OrderID"]);
        model.Code = Convert.ToString(reader["OrderCode"]);
        model.CustomerID = Convert.ToInt32(reader["CustomerID"]);
        model.Comment = Convert.ToString(reader["Comment"]);
      }
    }
    ```

</details>

# Adapter API

The basic API of the adapter pattern is a set of concepts and shapes that serve to 
- Clearly express what data storage actions are valid and [intended](#expressing-what-is-intended)
- Organize storage APIs in human-scale, [discoverable](#dot-discoverability) groups
- Insulate the runtime [data model](#model) from [storage implementation](#storage-adapter) details
- Facilitate [clear mechanics](#clarity-of-mechanics) throughout the storage logic
 
These concepts and shapes are language-independent, and can in fact be implemented with similar semantics in many different languages.
  
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

Many models have some attributes that are intended to be mutable and others that are immutable. Mutable attributes can be updated one or many times after the record is created, while immutable attributes are provided exactly once when the record is created, or even generated by the data store, and then should never be changed. "Should" another human concept that can only be known and expressed by a human.

When the language supports it, immutable attributes should be clearly expressed in the language-specific idiom:

<details open>
<summary><em>Example - Immutable Attributes</em></summary>

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

</details>

#### *What things are related?*

If I have an `Order`, what other models are directly related and reasonable to retrieve? ORM frameworks often express these through "relations", "navigation properties", or similar concepts. The point here is to include on the model only those related models or collections that are conceptually directly related. 

For example, the one *required* `Customer`, one *optional* `SalesPerson`, and zero to many `LineItem`s associated with an `Order` model are fundamentally part of the order concept itself. They should be present on / accessible directly from the model. Likewise, the one `Order` associated with a `LineItem` is part of the `LineItem` concept itself, and so should be accessible directly from the `LineItem`.

In contrast, the zero to many `Orders` for a `Customer` are **not** fundamentally part of the customer concept itself. The `OrderAdapter` (see [model adapter](#model-adapter) below) should provide a `FindAll` method for finding the `Orders` for a `Customer`, but the `Customer` model should not include an `Orders` collection.

Go
```go
func (o *Order) Lines() ([]LineItem, error)   { /*..*/ } // Yes
func (o *Order) Customer() (*Customer, error) { /*..*/ } // Yes
func (l *LineItem) Order() (*Order, error)    { /*..*/ } // Yes
func (c *Customer) Orders() ([]*Order, error) { /*..*/ } // NO!

type OrderAdapter interface {
  FindAllByCustomer(c *Customer) ([]*Order, error) // Yes
}
```

## Model Adapter

**Each** model type has a corresponding `Adapter` type which defines only and exactly the valid storage actions for that model type, and in an implementation-independent manner. When the language supports it, the model adapter type should be an [interface](https://en.wikipedia.org/wiki/Protocol_(object-oriented_programming)).

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

Depending on the preferences of the team, the ability of the language to express it, and the specific object model, the `Create` method on the model adapter for a particular model may be non-public, and instead exposed via a factory method on a related *model*. Both of the following are perfectly dandy. Just pick one and stick with it:

<details open>
<summary><em>Example - Create Method</em></summary>

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

</details>

### \*\* Non-Public Delete, Save

Depending on the preferences of the team and the ability of the language to express it, the `Delete` and `Save` methods may be hidden from the public API and instead exposed as instance methods on the corresponding model class. Both of the following are perfectly dandy. Just pick one and stick with it:

<details open>
<summary><em>Example - Update Method</em></summary>

C# - Update methods on Adapter
```C#
var con     = GetConnection()
var order   = con.Order.Get(oderId)
order.State = OrderState.Complete
con.Order.Save(order)
```

C# - Update methods on model
```C#
var con     = GetConnection()
var order   = con.Order.Get(oderId)
order.State = OrderState.Complete
order.Save()  // Calls internal method con.Order.Save()
```

</details>

### Neutral Semantics

A model adapter uses semantics that do not imply a specific kind of data store. They do not change if storage switches from a direct connection to relational database to a protobuf API over gRPC. So we `Get` (neutral) instead of `Select` (SQL), and `Save` (nuetral) instead of `Post` (HTTP). 

If you like `Retrieve` better than `Get`, or `Remove` better than `Delete`, or `Update` better than `Save`, or `Search` better than `Find` as implementation-neutral words for these ideas then go for it. Just be consistent.

### Singular and Plural Semantics
In the table above I used for example `Get` and `GetAll`. If you want to use `GetOne` and `GetList`, or `Get` and `GetMany` then thumbs up, just be consistent. 

### Overload Semantics
A single adapter may have multiple `Get`- and `Find`-type methods, corresponding to different well-defined inputs from which you might get or find one or more models. In languages that support overloading it makes sense to use the overload technique here instead of different method names. In dynamic languages either can be implemented, and it is a matter of taste which is "clearer". In other languages overloading is not possible at all and so different names must be used. Once again, pick a convention and stick to it.

<details open>
<summary><em>Example - Overload Semantics</em></summary>

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

</details>

### Only Include Required Actions

Some models, like log entries, are intended to be completely immutable. They should neither be updated nor deleted after they are created, except perhaps by some back end rotation mechanism implemented outside the application layer. In these cases, don't include a `Save` or `Delete` method on that model's adapter. 

User models are often updatable and can be permanently disabled but can't actually be deleted because it would destroy historical records. If this is the case in your system then don't include a `Delete` method for users.

In actual applications, most models are **not** normally *searched for*, and so should not have any `Find` or `FindAll` methods.

The generalization here should be clear: The point is to express and expose only the intended, reasonable, valid storage actions for each model.

## Connection

The root interaction with data storage is through a `connection` object. The connection object's public API does not reveal what kind of data store (or even multiple stores) that it connects to. When the language supports it, this should be an [interface](https://en.wikipedia.org/wiki/Protocol_(object-oriented_programming)) rather than a concrete class.

The main purpose of the connection is to be a gathering point for the individual model adapters. For a small application they may all be at the top level. For a larger application they may be grouped by concept, which may or may not correspond to an actual storage group or schema.

<details open>
<summary><em>Example - Connection API</em></summary>

Go - Top Level
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

C# - Nested
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

</details>

### Dot Discoverability

Code must be understood by humans. It is communication from one human to another in a form that also encodes actual mechanics that a computer can execute. As described at length above, the adapter pattern aims to aid understanding by making the number of options in any given situation both explicit and limited. For example, the list of "storage actions for a user" is explicit on the user model adapter, and is limited to what is reasonable, relevant, and intended.

The particular shape of the APIs described above is in part influenced by how we actually write code -- in editors and IDEs with moderate intelligence, which support auto-completion and inline documentation. In the vast majority of contemporary general-purpose languages, the period or dot (`.`) is the member accessor, and also the trigger for the editor to display a list of *things that can come next*. 

The particular semantics of `connection[.group].model.action` directly aids this by grouping the lists of what comes next into small steps, where the options at each step are limited and relevant. With two or three branch levels starting from `connection`, we can express even very large data models succinctly. 

In practice, this means a consuming developer (including oneself after enough time has elapsed to forget details), can discover a substantial amount about the modelled domain just by typing a few lines in a capable code editor.

# Implementation

To state the obvious, storage actually has to be implemented in some specific place or places. Regardless of the format and protocol, this requires code to bind to the specific implementation details and mechanics of the selected storage.

## Storage Adapter
Within the adapter pattern, this mapping code is implemented in the **storage adapter**(s) for each model type. Its role is to implement the platform-neutral API of the [model adapter](#model-adapter) through platform-specific interactions with the actual storage provider.

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

### Solutions
Humans are good at thinking, but bad at mechanical tasks. Machines are good at mechanical tasks, but can't think at all. And that is the irreducible challenge of implementing a data access layer. 

There are several options:

#### All by hand

The advantage here is the simplicity of tooling. There is no thing-making thing to configure, manage, debug, and maintain.

The other very important advantage is in clarity of the actual code. Without complex ORMs handling storage operations, the call stack from business logic to storage should be short and clear. The SQL should be present in the source code, as are the procedural mapping statements. When something goes wrong, we can see exactly what is happening.

##### Dumb bugs

The downside is not time but *mistakes*. Compared to all the other work involved in building an enterprise data application, writing rote SQL statements and mapping logic by hand is not time-consuming. 

Rather, compared to all the other work involved in building an enterprise data application, writing rote SQL statements and mapping logic by hand is one of the easiest ways to introduce bugs. Latent, gnarly, hard-to-find bugs.

When a bug is identified, the clear mechanics of hand-rolled code often help debugging to be a simple exercise. Unfortunately, mapping mistakes easily lead to bugs that are not noticed, or whose effects do not obviously point to a mapping problem.

A partial workaround is to write exhaustive unit tests for all basic CRUD operations...but the same mistakes that can be made in mapping implementation code can be made in the code that tests it. Further, the mapping tests are often little more that repeats of the *implementation* of the mapping. 

##### Ambiguity

Another downside is ambiguity of intent, which is exactly the [kind of thing we are trying to avoid](#clarity-of-intent). Ontologies, models, and storage implementations all include *human decisions*, such as about which attributes should be immutable. Implementing the mapping and SQL code requires being aware of and implementing these decisions. The problem is that, unless everything is meticulously commented, a reader of the code does not know whether the choices actually implemented are intentional, even when the storage-to-runtime attribute mappings are correct. 

- *Is it intentional that X column in the database is never loaded at runtime?*
- *Is it intentional that attribute Y is omitted from the UPDATE statement?*
- *Is it intentional that a column is nullable in the database but not on the model?*

#### Use an ORM

The purported advantage here is speed and reduction in mistakes. Whether through convention or configuration, SQL code and/or runtime model code is generate from one gold source (often the mirror SQL or runtime model, possibly with embedded metadata). So in general it reduces the number of places where mapping errors occur to a single configuration point.

The downsides are many. Most ORMs do not simply automate mapping, they instead interject themselves throughout all storage code, and even into the model and/or storage layer. For all the reasons described here and in [Applications Software](../concepts/applications-software.md), this is bad, frustrating, and unnecessary. ORMs are the poster child of thing-making things, that introduce immense complexity, and yet cannot match the quality of a human doing the same "simple" tasks. 

Another challenge with ORMs is that most generate behavior and SQL code at runtime rather than compile time. This makes debugging difficult, because the actual SQL statement exists nowhere in source code, and the actual mapping statements are dynamically organized and executed. [Active Record](https://guides.rubyonrails.org/active_record_basics.html) provides an extreme example, where basic CRUD operations tunnel through dozens of layers of call backs and evaluated code generated at runtime. 

#### Code generation by hand

What I mean here is write your own code generators that do only and exactly what you need them to do. As you could likely anticipate, this is the middle way that I advocate. 

The problem with code generators written by other people is that they either need to stick to a strictly limited set of options (*their* initial requirements) which almost never match your own, or they have to support lots of options, only one subset of which is relevant to you.  

When you write them yourself, you can and should stick to a strictly limited set of options -- the ones you need. At risk of equivocating, I'd compare tightly scoped custom-built generators to [jigs](https://en.wikipedia.org/wiki/Jig_(tool)), [dies](https://en.wikipedia.org/wiki/Die_(manufacturing)), [patterns](https://en.wikipedia.org/wiki/Pattern_(casting)) and [forms](https://en.wikipedia.org/wiki/Formwork) used when making physical buildings and machines. They effectively reduce the variance and mistakes from humans trying (badly) to do strictly mechanical tasks. But they are often arranged or built for a specific project, and only applied to specific tasks within that project.

In the same way, we can write code generators (or [linters](#automated-validation)) that each perform a limited task and only per the requirements of a specific project. 

Done well, this approach yields the following benefits:

- **There is no *runtime* dependency**. We are generating code, not behavior. The code does not know or care how it came to exist, and the potential complexity and problems with the code-generating thing is at least confined to authoring time. When actually executing and debugging, the mapping / generating library is not present.

- **The code looks like a human wrote it**. If you build a simple generator yourself, you can ensure the code it generates looks just like the other code in the same project. While you may want to label certain files or blocks with comments like "This section generated", it should [clear](#achieving-clarity) and consistent with the style and conventions of hand-written code around it.

- **The thing-making thing is kept on a short leash**. Because you are not trying to solve a whole *class* of code generating problems, and you control the code-generating code, you can (and should!) maintain discipline. Resist the desire to make your own code-generating framework. It is ok to copy and modify it from one project to the next -- this is how learning and evolution happen. 

In general, I reiterate a point [I make elsewhere](../README.md#building-is-not-inventing): Building a thing is not the same as inventing. Knowing how to make small, custom code generators is an example of applying a skill, not "reinventing the wheel." 

#### Automated *validation*

A related approach automates code analysis rather than code generation. Moderately simple parsers and linters can be built to validate sanity checks like the following:

 - An attribute is not mapped to more than one parameter
 - A column is not mapped to more than one attribute
 - The table name in the SQL is correct
 - The data type of a column is compatible with the runtime type of a model attribute

For full validation, there is no avoiding that human intent must be recorded somewhere, whether in a separate data file or in embedded metadata.

# Principles

## Clarity of intent

One of the primary goals of the adapter pattern is to allow the authors of the data access layer to clearly express which actions are valid and intended. 

This is very different from many ORM models, or non-relational equivalents, which simply transcribe SQL or other data store protocol semantics to the runtime language. In other words, ORMs let you do most or all actions on most or all models; they just provide mechanics to allow one to execute SQL-like actions while spelling in a runtime language. Thus they support complex dynamic queries, and generate a standard set of methods for most model types corresponding to all the CRUD actions. 

In contrast, the adapter pattern exposes only and exactly the data and functionality that is intended. Intention is a human idea, and must be expressed by one human (an author of the data layer) to another (a user or reviewer of the data layer--which might be the same human a few days or years later).

## Clarity of mechanics 

A software application is a digital machine. To design, build, and support one competently it is necessary to actually understand how it works. The patterns described here strive to maintain clear *mechanics* without compromising the clarity of the *semantics* (intent).

The primary technique for accomplishing both is to draw clear boundaries between problem sets. Often the internal mechanics (the how) are the semantics (the why) of a lower layer. For a detailed illustration, see [semantics and mechanics](#semantics-and-mechanics).

## Achieving clarity
 
"Clarity" is one of those desirable qualities of code, like "readability" or "simplicity", where everyone can agree it's great but no one can agree what it looks like. In the case of "clarity", I believe it is partly because people don't *clarify* for themselves or others whether they mean clarity of *semantics* (purpose, intent) or clarity of *mechanics*. 

It is often difficult to achieve clarity of both semantics and mechanics at the same time, especially if the scope of the problem being addressed is too large. Either we have a clear intent with obscured mechanics (i.g. "magic"), or we have intent lost in explicit and clear mechanics, or a mediocre muddle of both. At worst, we don't have clarity of either, because the authors did not have clarity in their own minds as to what problem they were trying to solve. I place most ORMs squarely in this camp, for reasons described throughout this document.

The situation however, is not hopeless. It doesn't even have to be hard. The solution lies in the very human skill of drawing clear boundaries at appropriate scales, scopes, and seams. 

The outcome is an alternation of clear semantics on the outside of one layer to clear mechanics on the inside. Those mechanics must solve a tightly focused problem by in turn relying on the clear semantics of another layer. 

This is actually possible to do, and it is actually helpful. It does however require the authors to [*actually know what they are doing*](../README.md#mastery-of-fundamentals). They must understand which problem they are trying to solve at each level, and why, and understand the purpose and application of the tools available to solve those problems. 

Hopefully an extended illustration derived from the [opening example](#by-example) will make the meaning of "clarity" clear.

<details>
<summary><em>Illustration - Semantics and Mechanics</em></summary>

1. Semantics of `Order.Lines`

   We are solving the problem of **communicating what things are related to an order model**. 

   The public api of `Order.Lines` is a simple property which returns a `IReadOnlyCollection<OrderLine>`. We have communicated that an order has a collection of lines, but that you can't modify that collection directly.

   ```c#
   public IReadOnlyCollection<OrderLine> Lines { get; }
   ```

   From the outside, we don't know or care *how* the lines are retrieved, cached or updated. 

2. Mechanics of `Order.Lines`

    We are solving the problem of **retrieving and caching the lines related to an order model**

    In the implementation, we have simple *mechanics* for lazy loading the  collection: 
 
    ```c#
    internal List<OrderLine> _lines;
    internal IReadOnlyCollection<OrderLine> _linesRead;
    public IReadOnlyCollection<OrderLine> Lines
    {
      get
      {
        if (_lines == null) _lines = connection.OrderLine.GetAll(this).ToList();  
        if (_linesRead == null) _linesRead = _lines?.AsReadOnly(); 
        return _linesRead;
      }
    }
    ```
 
    Specifically:
 
    - We memoize both the mutable collection (`_lines`) and a read-only view of  it (`_linesRead`)
    - If the mutable collection hasn't been initialized, we load it from the  connection using the clear *semantics* of the OrderLine adapter
    - If the read-only view hasn't been initialized, we safely try to create it  using the built-in `AsReadOnly` method, whose semantics are also obvious. 
 
    So the internal *mechanics* of the `Lines` getter rely on the external  *semantics* of 
 
    - The `connection` (`OrderLine` property) and `IOrderLineAdapter` (`GetAll (order)` method)
    - Linq's built-in `ToList` method
    - System.Collection.Generic's built in `AsReadOnly` method
 
    At this level we don't know or care how the connection has an  `IOrderLineAdapter`, how the order line adapter actually gets the order  line models from the data store, how `ToList` creates a list, or how  `AsReadOnly` creates a concrete thing that implements  `IReadOnlyCollection<T>`

3. Semantics of `IOrderLineAdapter.GetAll(Order)`

    We are solving the problem of **describing the finite list of valid, reasonable, intended storage actions for order lines**

    As just noted, the public API of this method is simple and semantically  clear. Give it an order, and it will get all the order lines associated  with that order. As an interface method, it doesn't even have an  implementation. We defer to whatever [composition root](https://freecontent.manning.com/dependency-injection-in-net-2nd-edition-understanding-the-composition-root/)  created the connection for us to supply an implementation of  `IOrderLineAdapter`.

4. Mechanics of `OrderLineSQLAdapter.GetAll(Order)`
 
    We are solving the problem of **mechanically retrieving all order line models for a given order**

    Following the pattern in the extended example, the actual mechanics of  `IOrderLineAdapter.GetAll(Order)` would be implemented in a  `OrderLineSQLAdapter`, which reuses shared mechanics from  `SQLAdapterBase<T>`:
 
    ```c#
    internal class OrderLineSQLAdapter : SQLAdapterBase<OrderLine>,  IOrderLineAdapter {
      public IEnumerable<OrderLine> GetAll(Order order) {
        return base.SelectAll(new { OrderID = order.ID });
      }
    }
    ```
 
    At this border / boundary / seam, we transition from the platform-neutral semantics of the `IOrderLineAdapter` ("Get" all) to the SQL semantics of the internal implementation of "Select" all. I trust these internal semantics are once again clear:  `base.SelectAll` will execute an appropriate `SELECT` statement against  some backing relational database, filtering for where column `"OrderID"` = [the value of `order.ID`]. 

    We also transition from the intentionally specific runtime semantics to the generic semantics of SQL. `IOrderLineAdapter.GetAll(Order)`) specifically accepts an `Order` object. We are communicating that it is a reasonable and valid action to get all the lines for the order, and by using 'Get' instead of 'Find', we are communicating that the lines for an order are a clearly defined set. 
    
    In contrast, `SQLAdapterBase<T>.SelectAll(object)` reflects its own purpose, which is to faithfully execute a `SELECT` statement using whatever parameters were provided. 

    Knowing that `GetAll(Order)` is a method that should be included on the `IOrderLineAdapter`, and knowing that it can be implemented with the one-liner `base.SelectAll(new { OrderID = order.ID })`, is the kind of "simple" human decision that exemplifies the work of building this layer.

5. Semantics of `SQLAdapterBase<T>.SelectAll(object parameters)`

    We are solving the problem of **communicating which mechanical storage actions are already implemented in the shared SQLAdapterBase**

    Because `SQLAdapterBase` can already `SelectAll` given arbitrary parameters, we know we don't need to re-implement these mechanics in specific model's storage adapter. 

    Any code which has knowledge of and access to the `SQLAdapterBase` already  knows full well that it is interacting with a relational database using  SQL. I trust the intent (semantics) of this method are abundantly clear:  `SELECT` all the records for the given model type using the provided  literal parameters.

6. Mechanics of `SQLAdapterBase<T>.SelectAll(object parameters)`

    We are solving the problem of **mechanically retrieving the actual data from a relational database**

    Finally we get to the standard rote mechanics of opening connections,  creating commands, and executing SQL statements:
 
    ```c#
    internal abstract class SQLAdapterBase<T> {
      // ...
      protected IEnumerable<T> SelectAll(object parameters) {
        // Convert whatever came in to a dictionary
        var dbparams = SomeUtil.ToDictionary(parameters)

        using(var sqlCon = GetSqlConnection()) {
          sqlCon.Open();
          var cmd = sqlCon.CreateCommand();

          // Build a basic filtered SELECT statement
          cmd.CommandText = this.SQLSelectBase + 
            SomeUtil.WhereSQL(dbparams.Keys);

          // Set the parameters
          foreach (var pair in dbparams)
          {
            cmd.Parameters.AddWithValue(pair.Key, pair.Value);
          }
          using (var reader = cmd.ExecuteReader())
          {
            // Stream the result set
            while (reader.Read()) {
              var model = this.Construct();
              this.ReadRow(reader, model);
              yield return model;
            }
          }
        }
      }
      // ...
    }
    ```

    The mechanics at this level rely on multiple lower levels:

    - Implementations of a relational database driver which know how to `Open` a connection, create a command, associate parameters with that command, execute it to create a cursor, and scroll through the results of that cursor. We simply rely on the *semantics* of the `System.Data.DbCommon` API. 
    - Some helpful utilities that turn things into dictionaries (`ToDictionary`) and build trivial `WHERE` clauses from a list of field names (`WhereSQL`)
    - The model-specific `SQLAdapter` to implement: 
      - `Construct` to instantiate a new model instance, injecting any required information as necessary
      - `ReadRow` to map column names to model properties and convert raw values returned from the database to appropriate runtime types.

At this point we could continue down the stack, but I trust the point is now clear: Drawing clear boundaries between layers, while alternating between clear semantics and clear mechanics, we can build an arbitrarily large digital machine that can be understood, maintained, and modified.

Because the specific logic at each level is often simple and obvious, we can also automate or generate *specific parts* as appropriate, without giving up the ability to step in with human decisions like implementing the simple `IOrderLineAdapter.GetAll(Order)` method.

</details>
 
## Independence of model and storage

As discussed at length in [Applications Software](../concepts/applications-software.md#principles), the work of designing a runtime data model and the work of designing appropriate storage for that model are related but separate concerns. Both follow from the [ontology](../concepts/applications-software.md#ontology-first), not from each other. When implemented well, they almost always have subtle (or substantial) differences. 

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

What remains is the task of mapping the model to the storage. This is necessarily human work, though portions of it can be automated. Regardless of whether it is built by a person or machine, the point is that what needs to be built is this:

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

In other words, the ontology is what drives both the runtime model design and the storage design, but the runtime model and storage design do *NOT* influence each other. It follows that they cannot be generated from each other, and yet that is exactly what most (all?) ORM frameworks do.

Because the runtime model and storage design can't actually be generated from each other, ORMs provide mechanisms to embed information about one of these concerns into the other, usually by embedding implementation details about the storage design in some form of inline metadata in the source code of the runtime model. See for example numerous mapping and serialization attributes in .NET ORMs, or struct tags used in Go ORMs and serialization libraries.

As a result, the runtime model and storage design become awkwardly but deeply dependent upon each other. At the same time they introduce complex mechanics to load the embedded metadata (usually through reflection) and generate behavior at runtime. This in turn clobbers the otherwise clear call stack from the outermost edge API to the actual storage action.

### Model != Row

A more specific version of the idea that models and storage designs are different things is that *models are not "rows"*. Some ontology concepts with a single model may in fact be stored in multiple tables, files, etc. This is exactly the kind of decision humans need to make when designing storage vs runtime models. 

For examples, see [Wide Records](#wide-records) and [Dynamic Records](#dynamic-records).

# Design Points

In addition to the specific [API](#adapter-api) and the [principles](#principles) described above, the adapter pattern includes a handful of important design points that distinguish it from other contemporary data access patterns, particularly those represented in most ORM frameworks.
 
## No unstored object graphs

This point is not required by the adapter semantics itself, but is a key technique for simplifying the mechanics of the data layer -- and making things easier for humans to understand and maintain. 

Building object graphs before they are stored is a major source of complexity in data layers, whether hand-rolled or through framework ORMs. I have built such mechanics myself more than once. It was a great puzzle and I felt very proud of my clever work, and I suspect many ORM authors and users also thrill at the delightful algorithms used to make sure everything is saved in the right order, or *possibly rolled back if something goes wrong part way through*.

##### C# : With an ORM that allows unstored object graphs
  ```c#
  customer = new Customer(customerName)          // Not yet in data store
  category = new ProductCategory("Coffee")       // Not yet in data store
  product  = new Product(category, "Dark Roast") // Not yet in data store
  order    = new Order(customer)                 // Not yet in data store
  repo.Save(order) // Automagically saves everything 
  ```

It is not worth it. Period. Do not do it. Instead, provide `Create` actions that 
 - Express exactly what inputs are required to store a *valid* record of the applicable type.
 - Require valid models (not model IDs) for required related models
 - Only return a model if the storage action (i.e. `INSERT` in SQL) succeeds 

This implies something else, too:

> In languages that support it, model constructors should be non-public

You don't ever *want* a runtime instance that doesn't correspond to a record that is or at least was in the data store. One way to **communicate this intention** is by making the constructors on model classes non-public, and instead only providing model instances through the corresponding adapter's `Create` or `Get/Find` methods. 

## No automatic collection synchronization

Like fancy model graph saving, collection synchronization is a major source of complexity in ORM frameworks and of confusion and woe for developers, and also leads to ambiguous semantics. 

  C#:
  ```c#
  order.Lines.RemoveAt(1) // Did this just delete a line?
  order.Save()            // Maybe this did?
  con.Save(order.Lines)   // Or do I need to do this? Or something?
  ```

Ideally, there should be exactly one way to add an item to a collection, and one way to remove it. I advocate the following approach:

- The canonical way to add an item to a collection is to call a factory method on the parent model (see [`Create` method](#-non-public-create) below). This method is responsible both for saving the new record in the data store and adding its runtime model to the parent model's runtime collection
  
  Go:
  ```go
  // Inserts a new line, and adds it the order's runtime slice of lines
  line, err := order.AddLine(product, quantity)
  ```

- The canonical way to remove an item from a collection is to simply call its `Delete` method. This method is responsible for removing the model from its parent model's runtime collection, IF the item model has a reference to its parent model. References should almost always be lazy-loaded, so a model that may have a parent in the data store might not yet have a reference to a corresponding parent *model* at runtime.

  C#: With Parent Reference
  ```c# 
  var line = order.Lines[2]  
  line.Delete()
  order.Lines.Contains(line) // false
  ```

  C#: Without Parent Reference <span id="delete-collection-item">&nbsp;</span>
  ```c#
  var line  = con.LineItems.Get(2312)
  var order = con.Order.Get(line.OrderID)
  console.WriteLine(order.Lines.Count)  // Cause Lines to be loaded
  line.Delete()
  order.Lines.Any(l => l.ID == line.ID) // true

  // order.Lines was loaded while the line was still in the data
  // store, and line above had not loaded its parent record and so
  // did not remove itself from any parent collection
  ```
 
## No automatic object graph updates

The two points above about unstored object graphs and collection synchronization are really two special cases of a more general problem: Attempting to automatically save object graphs. 

In short, *what is the effect of doing this?:*

```C# 
// Does this also save order.Customer?? order.Lines?? order.Lines[1].Discount??
order.Save()  
```
 
I repeat: Don't do it. It's not worth it. In introduces a great deal of complexity and hides mechanics, two cardinal sins in applications software engineering.

While I present various ok options in some sections below, here I advocate strongly for only one approach:

- Each model type has its own `Save` or `Delete` method
- A model's `Save` method updates only the updatable attributes represented on the model itself
- The only way to save or delete a model is through its own `Save` or `Delete` method

### Saving Graphs

This means that saving an object graph must be done explicitly. If it is important to data layer authors to provide the convenience of saving a whole graph starting from some well-defined origin, then they can define methods which clearly communicate what is happening, and with an implementation than can be viewed directly in the source code:

Go:
```go
// SaveGraph saves the order, its lines, and all associated *transactional* records such as discounts applied, taxes, shipping, etc.
func (o *Order) SaveGraph() error {
  if err := o.Save(); err != nil {
    return err
  }
  for _, line := range o.Lines() {
    if err := line.SaveGraph(); err != nil {
      return err
    } 
  }
  // TODO: Save order adjustments
  // ... etc.
}
```

## No Automatic Reference Equality

The [last example](#delete-collection-item) in the discussion of [collection synchronization](#no-automatic-collection-synchronization) illustrates another point: The connection should not be responsible for automatically guaranteeing reference equality for models. In other words, *if you retrieve the same record twice, you should expect to get two different model instances*:

JavaScript:
```javascript
let order1 = con.Order.Get(1)
let order2 = con.Order.Get(1) 
order1.ID    === order2.ID    // true  : Loaded from same record
order1.State === order2.State // true  : Loaded from same record
order1       === order2       // false : Not the same instance

// Mutation on one instance is not visible on the other
order1.State = OrderStates.Complete
order1.State == order2.State // false
```

Guaranteeing reference equality is yet another source of great complexity, because it requires connection-level caching to retain maps between primary or natural keys and loaded references, which in turn requires taking on the irreducibly difficult problem of cache synchronization and invalidation.

Sometimes reference equality is helpful for compact in-memory representation (we don't want 5,000 instances that all represent the same record). Treatment of this problem is covered below in Caching.

## Wide Models

I've written at length about the distinction between [model and storage](#model--row). 

One common example is very wide records that have both validated required attributes (conceptual foreign keys) and many optional attributes. Depending on the scale and on the specific storage types and products selected, it may be desirable to split the storage of these records into multiple tables or even multiple data stores. For example, the traditional relational / validated attributes may be stored in a single table in a main relational database, while other attributes are stored:
 
 - In validated or unvalidated JSON in the same relational database
 - In multiple validated psuedo-column store tables within the same relational database
 - In a separate validated or unvalidated column store, document store, or key-value store

It's entirely possible that these implementation decisions may even change as actual use patterns emerge or change, new features become available on storage products, or outside business forces motivate a change in storage products. 

The point is the *none of this is visible on the model or model adapter*. If a wide ontology concept is represented with a single model, then `wideThing.Save()` saves all of the attributes represented on the runtime model. 

If the data layer authors want to communicate the difference in character and possibly cost between the primary and extended attributes, then this difference should be made explicit in the runtime model, such as by partitioning attributes into child model parts:

<details open>
<summary><em>Example - Model Parts</em></summary>

C#
```c#
order.Discount  // Attributes related to discount
order.Return    // Attributes related to return 

// OrderDiscount and OrderReturn are model parts, not models.  They do NOT
// have their own adapters. Instead, they have explicit save methods via
// Order model or Order model adapter:
con.Order.SaveDiscount(order)

// OR
con.Order.Discount.Save() // calls internal con.Order.SaveDiscount()

// OR
con.Order.SaveDiscount()  // calls internal con.Order.SaveDiscount()

// OR
con.Order.Save(OrderPart.Base | OrderPart.Discount | OrderPart.Return)
```

</details>
 
## Dynamic Records
A special case of the wide record is dynamic records, whose available attribute keys and values may be completely open-ended, or limited by a *data-validated* set rather than a *schema-validated* set. If the entire record is dynamic and unvalidated, then this is an obvious use case for using a document store. 

Imagine for example a logging application which stores serialized HTTP requests. Metadata about these requests need to be validated and indexed in a main relational database, but HTTP headers (which have no set schema) will be stored in a document store, and body streams in separate blob storage. 
 
It is up to data layer authors to create models and model adapters that clearly communicate concepts like "these attributes are dynamic", or "retrieving the body stream is expensive" *without* exposing storage implementation details like:
 - "The body is stored in a blob storage" vs "The body is gzipped and stored in a MS SQL varbinary column"
 - "The headers are stored in as a JSON *string* in the relational database" vs "the headers are stored in a Postgres JSONB column" vs "the headers are stored as documents in MongoDb". 

<details open>
<summary><em>Example - Neutral semantics for a dynamic model</em></summary>
 
C#
```c#
// Storing a new request
var user     = CurrentUser();
var req      = context.Request;
var reqModel = con.Request.Create(user, req); 

// Iterating the dynamic headers collection of the model
foreach(var entry in reqModel.Headers) {
  Console.WriteLine($"{entry.Key}: ${entry.Value}")
}

// Searching for a request by a specific header value
var requests = con.Request.FindAll(
  headers: new Map {{ "X-Code-Name" , "bluebonnet" }}
)
var reqModel = requests.FirstOrDefault()

// Retrieving the body stream
reqModel.GetBody()
```

</details>
 
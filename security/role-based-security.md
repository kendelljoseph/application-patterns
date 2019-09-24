# Pattern Brief: Role-Based Security

Given:

- Data is valuable
- Deleting or corrupting data destroys value
- Disseminating data may constitute a breach

Then:

- Data itself should be secured

## Concepts

- A **principal** (or **user**) is an entity requesting or attempting to access some data or perform some action
- A **privilege** describes a specific thing that may be done ("read invoices", "update security"). A privilege ALWAYS is or includes a verb
- A **scope** is a data-dependent context which may limit a privilege. (Update security *only in this group*, create invoices *only for this customer*)
- A **role** is a semantically-named *kind of user*. ("Field Engineer", "System Admin")
- A **group** is a semantically-name *set of users*. ("Cleveland Service Team")
- A **gate** is a place in code where **authorization** is checked, and an action is allowed to proceed or is rejected

### Trivialization

All of the above concepts always exist, whether they are explicitly modeled or not. Some very simple systems can "eliminate" one or more of the above concepts by implicitly trivializing them:

- The trivial **privilege** is "can read/write/do all the things"
- The trivial **scope** is "any time, to any thing in the system"
- The trivial **role** is "people and/or machines that use this system"
- The trivial **group** is "all the users"
- The trivial **gate** is "logging in" (authentication)
- **Principal cannot be trivialized**. There can be no authentication without identity, and no security without authentication. Principal might not mean a specific human. E.g. the principal might be "the (and thus any) entity that knows a certain secret / possesses a certain certificate"

### How the things get connected

- **Privileges** are assigned to **roles** ONLY
  - A role represents a semantically meaningful collection of semantically meaningful privileges. 
  - Privileges are a **data concept**. 
  - Privileges are a **fixed list** known at **compile time**.

- **Users** are assigned to **groups** and/or **roles**.
  - **Users** are never assigned directly to privileges. 
  - Users are a **data concept** loaded from a data store at **run time**. 

- **Roles** are assigned to **users** and/or **groups**
  - Roles are a **data concept** loaded from a data store at **run time**. 

- **Scopes** are associated with a specific **role** assignment ONLY
  - Roles themselves are not scope-specific. A "Office Manager" must have the same set of basic privileges regardless of *which* office (a scope) they may be assigned to. 
  - Scopes are a **data concept** loaded from a data store at **run time**. 
  
- **Gates** inquire about a **principal**, a **privilege**, and a **scope**. The privilege can and should be hard coded. 
  - **Gates** NEVER inquire about roles. "Being an administrator" is not a privilege. "Creating a user" is a privilege.
  - Gates are a **procedural construct** implemented in source code
  - Gates do not contain the logic for *answering* the IsAuthorized question. They *ask* the question by supplying the correct parameters for the context.
    ```javascript
    var user    = CurrentUser()
    var context = GetContext(user)

    // This is a gate. It *asks* whether the use can ReadSecurity in the current context
    if (security.Can(user, Privilege.ReadSecurity, context)) {
      /* Do things */
    }
    ```

#### In natural language

The following table presents examples illustrating the concepts. They do not represent the actual object model of any particular real application.

|Concept|References|Example|
|-|-|-|
|Principal|`none`|Malcolm Doherty is a human user identified by `mdoherty`|
|Privilege|`none`|`AddEmployee` is the privilege to add a new human user to the system|
|||`ReadCalendar` is the privilege to view calendar items|
|||`ReadPosts` is the privilege to read topical posts published in the system|
|Role|`none`|`OfficeAdmin` is a human who manages a branch office|
|||`OfficeMember` is a human who works at a branch office|
|||`Employee` is a human who works for the organization|
|Group|`none`|`ClevelandTeam` represents all the humans who work at the Cleveland office, regardless of their role|
|||`Humans` are all human users of the system|
|Scope|`none`|`Office:Cleveland` is the abstract security container referenced by the `Office` record for Cleveland|
|Role-Privilege|Role, Privilege|`OfficeAdmin`s have `AddEmployee` privilege
|||`OfficeMember`s have `ReadCalendar` privilege|
|||`Humans`s have `ReadPosts` privilege|
|Group-Principal|Group, Principal|Malcolm Doherty is in the `ClevelandTeam` group| 
|||Malcolm Doherty is in the `Humans` group| 
|Role-Group[-Scope]|Role, Group [, Scope]|The `Humans` group has role `Employee` (unscoped / global)
|||The `ClevelandTeam` group has role `OfficeMember` in scope `Office:Cleveland`
|Role-Principal[-Scope]|Role, Principal [, Scope]|Malcolm Doherty is has role `OfficeAdmin` in scope `Office:Cleveland`

**Effective Permissions**

The above configuration resolves to the following privilege list for Malcolm Doherty. He can:
- `ReadPosts` throughout the system (via group membership and role assignment to group)
- `ReadCalendar` for events associated with the Cleveland Office (via group membership and scoped role assignment to group)
- `AddEmployee`s to the Cleveland Office (via scoped role assignment to user)

## Synchronization with directory services

It is often desirable to link user principals to existing identities in an external directory service, and also to delegate authentication of those users to the directory service. The local data store must store a record of some kind representing the user, including whatever unique identifier the directory service exposes to unambiguously identify the user. Ideally this is a **GUID** which does not change even if the users' user name or email change. People change their names and email addresses! So use something non-semantic and permanent (like a GUID if available) to identify them if at all possible. 

Sometimes organizations also want to synchronize *groups* with external directory services. This can greatly simplify cross-organization rights maintenance, as a single directory group can be created for a specific job function, and people can be added or removed from that group as necessary. If systems access is tied directly to that group membership, there is no chance to forget to add or remove those people from numerous disconnected systems to which they may need (or no longer need) access.

In these cases, the identities of relevant external groups should be stored in the local data store, but

> **BE VERY CAREFUL ABOUT REPLICATING GROUP _MEMBERSHIP_ IN THE LOCAL DATA STORE**

When access is tied to group membership, then it is critical that the group membership information be correct. Copying it means synchronizing it, and that introduces potential for errors in synchronization. At worst, a user will continue to have access that was intended to be revoked. 

It can be helpful to have group membership stored locally, as it allows the security within the data store to be complete (you can determine a user's authorization rights from the data store alone). *If* this is done, care should be taken to guard against failed synchronization, such as storing on the user record the last time group membership was refreshed. It should also be possible to force refresh the cached group membership for a user or a group.

## Modularization and decoupling

### Security evaluator

Gate **evaluation** should be implemented in **exactly one place**. The inputs are again a principal, a privilege, and a context which includes scope(s) if applicable. The **security evaluator** can access all necessary information to apply any business rules (most basic is role membership) to determine if the given principal has access to the requested privilege and scope.

The security evaluator may retrieve configuration information (this is its purpose), but it may **not** add dynamic data context. Examples would be the current time or the current location. This couples (in this case) environment sensing with security evaluation. If access is to be limited by date range or time of day, then the gate must supply the effective timestamp in the context supplied to the evaluator. This also allows testing and inquiry -- you are free to ask if a principal *will* or *did* have access at any specific time, given the current configuration.

#### Security vs non-security rules

Business rules determining when or how actions should be performed should **NOT** be implemented in the security evaluator. The security evaluator determines whether an action is allowed from a authorization perspective only, not whether it is valid or correct per business rules. Other logic within the application layer is responsible for determining whether an action is correct and valid, not whether a particular principal is authorized to perform that action.

||Security Rule|Business Rule|
|-|-|-|
|Validates|Authorization|Correctness|
|Rule Failure Causes|Breach|Mistake| 

#### Validation Order
Within the life cycle of a request, validation generally occurs in the following order:

- Security gate at API layer (optional, early response when invoking a particular API action)
  - Answers: Would this ever be allowed? *Does the user have XX privilege in any scope?*
  - An incorrect acceptance here may leak some information *about* the application, such as which parameters are accepted. 
  - A correct rejection here represents an authorized access attempt.
  - Correct response on rejection here returns minimal information to client ("403 unauthorized")
- Input validation to API layer (validate deserialization and presence of protocol inputs, map to internal application layer inputs)
  - Answers: Would this ever be correct? *Is that a real product ID? Were all the required parameters supplied?*
  - An incorrect acceptance here will lead to invalid data being forwarded to the application layer
  - Correct response on rejection here provides useful information to client ("400: Missing required parameter 'ProductCode'")
- Security gate in application layer (required. The firm accept/reject decision for the action in general)
  - Answers: Is this allowed? *Can the user perform the requested action given the provided context?*
  - An incorrect acceptance here constitutes a **breach** (unauthorized action incorrectly allowed) or a **block** (authorized action incorrectly rejected)
  - Correct response on rejection here provides useful information to client ("403: User xx not authorized to read security")
- Business rule validation in application layer (implementation of business rules)
  - Answers: Is this correct? 
  - An incorrect acceptance here may allow *incorrect* data. E.g. a withdrawal over the daily limit was allowed
  - A correct rejection here represents an error in user input (not a bug)
  - Correct response on rejection here provides useful information to client ("400: Withdrawal amount 3,200 exceeds daily limit 2,500 for account XYZ")
- Data validation in data store layer (evaluation of constraints such as uniqueness and foreign keys)
  - Answers: Is this possible?
  - An incorrect acceptance here may allow *invalid* data. E.g. a line item references an order that doesn't exist
  - A correct rejection here represents an error in the application code. Application code should validate inputs before attempting to store them, even though the data store still provides the *guarantee* of validity
  - Correct response on rejection here provides minimal information to the client ("500: Unexpected server error"), while internal details should be logged
 
### Symbol validation, hard coding
The list of privileges must be known at compile / build time. Therefore the list of possible privileges should be in code, and that code should be referenced symbolically by gates. This can be done in any language. A dynamically typed language like Ruby or JavaScript can use named constants, while many languages provide an enumeration concept.

If it is desired that security will be highly *configurable* as data, privileges may be simplified while scope will be made more complex. For example, perhaps privileges are simply "Read", and "Update", but the scope includes "Object type" and "folder path", both of which are modifiable data stored in the data store. 

### Principals are all evaluated the same
It is common for there to be at least two kinds of principals -- humans and machines. Humans *users* are often linked to an external directory service, which also handles password-based authentication. Machine users are often authenticated internally by the application itself (such as by an API token) and represent automated services or software components interacting with the application. 

Even in these cases, there should be exactly one implementation of the gate evaluator. You do not want to build one set of *mechanics* for one type of principal and different set of *mechanics* for a another.

### In a relational database, there should be one principals table

In order for principals to be treated the same, they must all be stored in the same table in a relational database. This allows for sane validation with normal foreign keys and constraints. If it is desired to store data about different kinds of principals in different tables, then these tables should have non-nullable foreign keys to the principals table. (e.g. each User has a Principal, and each SystemUser has a Principal).

If data about principals is stored in multiple tables, it is up to the author and the available language capabilities to determine whether or how these underlying storage mechanics are exposed on the runtime objects. E.g. in C#, Java, or TypeScript, one could have User and APIUser classes which inherit from a common Principal base class; separate User and SystemUser classes which both implement an interface; separate User and SystemUser classes which have their associated Principal as a property ... etc etc. 

Given the mechanical complexity introduced with using multiple tables for principals, it is preferable to use a single table (`Users` or `Principals`) that has columns to indicate the type of the user and their other optional properties. This could be as simple as a boolean `IsSystem` column. If it is important to enforce constraints for some types of users but not others (e.g. human users must have a `ActiveDirectoryGUID`), most databases support filtered check constraints that would accomplish this. e.g.

```SQL
-- Requires ActiveDirectoryGUID only if user is not a system (machine) user
CONSTRAINT CHK_HasADGuid CHECK (IsSystem = 1 OR ActiveDirectoryGUID IS NOT NULL)
```

### One Scope Table
For the same reasons of mechanical simplicity, there should be one Scope table, whose schema does not change regardless of the design or behavior of the application's scoping rules. Scope may be associated with multiple different kinds of things (a "Company" has a scope, a "Folder" has a scope, a "Product Line" has a scope, etc), and this can be evolved or changed over time without changing the core security object model. Any object that represents a scope has a non-nullable foreign key to the Scope table. 

## API Tokens

If the application supports API-based authentication with a shared secret, it should be possible to have more than one valid secret per principal. Automated or manual secret rotation always involves latency, sometimes of days, as new secrets must be deployed to client systems. At the very least, there must be overlap between when the new secret is *known* (but not necessarily activated yet) and when the old secret is still valid. 

Further, it may be wise to permanently retain old secrets, and raise an alert if they are ever submitted. An attempt to access a secured service using a naive wrong secret represents a very different security situation than if a client, whether through malice or ignorance, has possession of and is attempting to use an actual secret that is expired.

If at all possible, API tokens should not be persisted in the application data store. Even if they are stored in the same *kind* of data store (e.g. just another relational or document database), the credentials to access them should be secured and maintained separately from credentials for accessing the application data store. A better option is to use a dedicated secret store such as Azure Key Value, AWS Key Management Service, or Google Cloud Key Management Service and retrieve API Keys at runtime. In this case, Keys should be fetched on demand and cached for a limited period of time. 

*To be covered in another topic: Permanent shared secrets should in turn only be used to generate temporary tokens*


## Ontology 

AS SQL

```SQL 
CREATE TABLE "User" (
    Id          INT NOT NULL PRIMARY KEY
  , UserName    TEXT(50) NOT NULL
  , IsSystem    BIT NOT NULL DEFAULT(0)
  , ...    
)
 
CREATE TABLE "ApplicationPrivilege" (
    Id          INT NOT NULL PRIMARY KEY
  , Code        TEXT(50) NOT NULL
)

CREATE TABLE "ApplicationRole" (
    Id          INT NOT NULL PRIMARY KEY
  , Code        TEXT(50) NOT NULL
)

CREATE TABLE "SecurityScope" (
    SecurityScopeID      INT NOT NULL
  , Name                 TEXT(50) NOT NULL   
  , ...
)

-- Assign privileges to roles
CREATE TABLE "RolePrivilege" (
    ApplicationRoleID       INT NOT NULL
  , ApplicationPrivilegeID  INT NOT NULL

  , PRIMARY KEY (ApplicationRoleID, ApplicationPrivilegeID)

  , CONSTRAINT FK_RolePrivilege_Role
    FOREIGN KEY ( ApplicationRoleID )
    REFERENCES "ApplicationRole" ( ApplicationRoleID )
    ON DELETE CASCADE 

  , CONSTRAINT FK_RolePrivilege_Privilege
    FOREIGN KEY ( ApplicationPrivilegeID )
    REFERENCES "ApplicationPrivilege" ( ApplicationPrivilegeID )
    ON DELETE CASCADE 
)

-- Assign users to roles
CREATE TABLE "UserRole" (
    UserID              INT NOT NULL
  , ApplicationRoleID   INT NOT NULL
  , SecurityScopeID     INT NULL -- Nullable foreign key. Null represents global role
  
  , CONSTRAINT FK_UserRole_User 
    FOREIGN KEY ( UserID )
    REFERENCES "User" ( UserID )
    ON DELETE CASCADE

  , CONSTRAINT FK_UserRole_Role
    FOREIGN KEY ( ApplicationRoleID )
    REFERENCES "ApplicationRole" ( ApplicationRoleID )
    ON DELETE CASCADE

  , CONSTRAINT FK_UserRole_Scope
    FOREIGN KEY ( SecurityScopeID )
    REFERENCES "SecurityScope" ( SecurityScopeID )
    ON DELETE CASCADE
)

CREATE UNIQUE INDEX UQ_GlobalUserRole
  ON "UserRole" ( UserID, ApplicationRoleID )
  WHERE SecurityScopeID IS NULL
;

CREATE UNIQUE INDEX UQ_ScopedUserRole
  ON "UserRole" ( UserID, ApplicationRoleID, SecurityScopeID )
  WHERE SecurityScopeID IS NOT NULL
;
```

##### Attribution

Author: Joshua Honig. Copyright 2019 Mallowfields LLC. See [LICENSE](../LICENSE.md).

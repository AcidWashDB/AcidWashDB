# AcidWashDB

_*AcidWashDB does not yet exist. All statements here are aspirational.*_

AcidWashDB is an abstraction layer that provides ACID transactions
in the absence of failure and that degenerates to "eventually ACID" in the
presence of failure.

## Eventually ACID

AcidWash's "eventually ACID" consistency model makes the following guarantees:

* Database state converges toward a single authoritative history of ACID
  transactions.

* In the absence of failure or unusual propagation delays, each client's
  working state is a recent version of the authoritative history.

* In the presence of failure or unusual propagation delays, a client can choose
  to see a database state that includes not-permanently-committed transactions.
  In that case, the client can see which data is affected.

* A client can choose to make progress when it is on the minority side of a
  "partition" failure. This includes a disconnected client and a client that
  can only communicate with a minority of the storage servers.

* If a client performs work while on the minority side of a partition, as much of
  that work as feasible is automatically added to the authoritative history
  after the failure is healed. Transactions that cannot be automatically
  incorporated into the history (because they conflict with others that were)
  are presented back to the client. The client can choose to redo those
  transactions or to handle their failure.

## Data Model

* Cluster: a set of AcidWash services (stewards and royals) that collaborate
  to store data in a single Scylla cluster.

* Database: a configuration and naming domain within a cluster.

* Realm: a logical database partition -- the unit of horizontal scaling and the
  unit of data replication to clients. A realm is intended to be a cohesive set
  of data associated with a real-world entity, such as a user or a customer
  organization.

  Transactions within a realm are cheap, but cross-realm transactions
  have additional cost and latency comparable to
  (the cost of a small single-realm transaction) * (the number of realms involved).

  Each client replica of a given database subscribes to a list of realms. A
  disconnected client (a client that cannot access any storage servers) can only
  make progress within those realms. It can no longer read or write other
  realms. A client that can access any single storage server can continue to
  make progress across all realms.

  Any given realm is limited to the transaction throughput that a single royal
  can support. (We will have to determine this limit experimentally, but it
  should be at least 100 TPS.) If a database cannot naturally be divided into
  realms, but also does not need more throughput than a single royal can
  support, it is fine for that database to have just one realm.

* Table: a set of rows that share a schema (an ordered set of typed fields)
  and indices. A table can have multiple indices. An index can include multiple
  fields. A table can have at most one "primary" index and any number of
  "secondary" indices, although this distinction is only present to support SQL
  client replicas and makes no difference in native operation.

AcidWash uses [Cap'n Proto](https://capnproto.org/) schemas. A schema is
defined for a table when it is created. The schema can be changed, but only
in ways that are backward-compatible in two senses:

* Clients that were written against older versions of the schema must
  see new rows as valid data, whether or not those clients are recompiled
  with new versions of the schema. This is backward compatibility
  [as defined by Cap'n Proto](https://capnproto.org/language.html#evolving-your-protocol).

* Clients that are written against newer versions of the schema must see
  old rows as valid data. This places the following additional requirements
  on schema evolution:

  * All new fields must have default values.

The only way to make a non-backward-compatible schema change is to introduce
a new table with the desired schema and then eventually sunset the old table.

## SQL-Based Clients

For some platforms, AcidWash packages are available that manage client working
state in a relational database, with the relational schema being a simple
transformation of the AcidWash schema. Client software must still perform
inserts and updates through the AcidWash client API, but can perform reads
directly from the SQL database and (if supported by the SQL engine) can
register with the SQL engine for change notification.

The following platforms are priorities for implementing this functionality:

* browser-side JavaScript

* a Rust server (intended for replicating an entire AcidWash database in a SQL
  database like RedShift for analytics)

* iOS Core Data

## Design Notes

### Relationship Between AcidWash Data Model and Underlying ScyllaDB

AcidWash uses [ScyllaDB](http://www.scylladb.com/) as an underlying, primitive
persistence engine. The user's AcidWash data model is not reflected directly as
a Scylla data model. Rather, AcidWash supports its user-facing functionality
through its own use of Scylla.

### Node Types and Coordination

An AcidWash client connects to a "steward" service. The stewards in an AcidWash
cluster are interchangeable except for performance differences due to network
proximity. Stewards do not access Scylla directly; instead, they connect with
"royal" services through [Nanomsg](http://nanomsg.org/). Each steward maintains
contact with as many royals as it is able to reach at any given time.

The stewards and royals collaborate through LWT transactions to crown a "king"
of each active realm. Each king holds the throne for a specific
transaction-timestamp interval, typically O(1 second). (Everyone should be king
for a second.) The steward routes each realm-specific operation to the king of
that realm. To execute an operation when there is no current king,  a new king
is crowned. For a configured interval after a king was active (probably O(10)
seconds), the most recent king is the preferred new king. If a new king is
needed after that interval elapses, the royal with the fastest network
connectivity to the requesting steward is the preferred new king.

### Transactions

AcidWash maintains a transaction "log" in Scylla for each realm of each database.

#### Committing Single-Realm Transactions

The king of a realm attempts to commit a single-realm transaction as follows:

* Check in the transaction log whether any of the rows read, gaps observed,
  or rows modified by the transaction changed since the realm version that was
  used as input for the transaction. If so, process the transaction as a
  roll-back.

* Durably insert a transaction record, which contains the following
  * a transaction id (a timestamp plus a UUID)
  * the client id of the originating client (a UUID)
  * a transaction description domain identifier (a UUID)
  * a transaction description (a blob that is only meaningful to clients
    who understand the description domain)
  * the transaction id of the rolled-back transaction (if any)
    that this transaction is replacing
  * the transaction state: committed & updates not yet performed
  * the last transaction id of the realm version that was used as input for this
    transaction (must be a committed transaction)
  * the realm version that will result from this transaction
  * a list of all rows read and gaps observed by the transaction
  * and a list of all (row, column)s modified and inserted by the transaction,
    with old and new values

* Write new values to all updated (row, column)s.

* Mark the transaction log record as committed.

#### Rolling Back a Single-Realm Transactions

When a single-realm transaction rolls back, nothing has yet been persisted.
Instead of writing the transaction record to the transaction log, the king
writes it to a roll-back log. When (if) the king is next able to communicate
with the originating client, the king will inform that client that the
transaction was rolled back. The client may choose to originate a new
transaction to replace it.

If the originating client does not replace the transaction within a
configurable interval, the king will begin reporting the rolled-back
transaction to all clients who understand that description domain, to
give them an opportunity to replace it.

The king will only implement one replacement transaction for a given
rolled-back transaction.

### Committing a Multi-Realm Transaction

A multi-realm transaction is implemented as two-phase commit across the
transaction logs of affected realms:

* The steward chooses one of the affected realms at random as the coordinator.
  That realm's king becomes the "coordinating king".

* The coordinating king processes the portion of the transaction that affects
  its own realm, as described above for *Committing a Single-Realm Transaction*,
  except that new values for modified/inserted rows are not yet written and the
  resulting transaction state is prepared. If this portion of the transaction
  rolls back, the whole transaction roles back.

* The coordinating king then works with the kings of the other affected
  realms to process their transactions into a prepared state (or roll back).

* Once all affected realms are prepared, the coordinating king moves its own
  transaction state to committed as coordinator.

* Then the coordinating king works with the kings of the other affected realms
  to move their transactions to the committed state.

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

* Realm: a logical database partition, the unit of horizontal scaling, and the
  unit of data replication to clients. A realm is intended to be a cohesive set
  of data associated with a real-world entity, such as a user or a customer
  organization.

  Joins and ACID transactions are supported only within realms.

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

# Design Notes

AcidWash uses [ScyllaDB](http://www.scylladb.com/) as an underlying, primitive
persistence engine. The user's AcidWash data model is not reflected directly as
a Scylla data model. Rather, AcidWash supports its user-facing functionality
through its own use of Scylla.

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

Each transaction record specifies the version of the database that was read when executing that transaction and all data rows that were examined during transaction execution. This information is used for determining when transactions cannot be added to the authoritative history because of serializability conflicts.

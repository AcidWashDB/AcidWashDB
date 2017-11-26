# AcidWashDB

I envision AcidWashDB as an abstraction layer that provides ACID transactions
in the absence of failure and that degenerates to "eventually ACID" in the
presence of failure.

## Eventually ACID

AcidWash's "eventually ACID" consistency model makes the following guarantees:

* Database state converges toward a single authoritative history of ACID
  transactions. In the absence of failure or unusual propagation delays,
  every client sees some very recent version of this authoritative history at
  all times.

* In the presence of failure or unusual propagation delays, a client can choose
  to see a database state that includes not-permanently-committed transactions.
  In that case, the client can see which data is affected.

* A client can choose to make progress when it is on the minority side of a
  "partition" failure. This includes a disconnected client and a client that
  can only communicate with a minority of the storage servers.

* If a client does work while on the minority side of a partition,
  as much of that work as feasible is automatically added to the authoritative
  history after the failure is healed. Transactions that cannot be automatically
  incorporated into the history (because they conflict with others that were)
  are presented back to the client. The client can choose to redo those transactions
  or to handle their failure.

## Replication Realms

The data in an AcidWash database is divided into "realms" -- subsets of the
data that are commonly accessed together. For example, the data for a given
user or a given customer business might be placed in a realm.

Each client can choose a set of realms that it will replicate locally.
A disconnected client (a client that cannot access any storage servers) can
only make progress within those realms. It can no longer read or write other
realms. A client that can access any single storage server can continue
to make progress across all realms.

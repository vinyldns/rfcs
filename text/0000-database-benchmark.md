_NOTE: This format comes from the
[Rust language RFC process](https://github.com/rust-lang/rfcs)._

- Feature or Prototype: database_benchmark
- Start Date: 2018-08-15
- RFC PR: (leave this empty, populate once the PR is created with a github link to the PR)
- Issue: (once approved, this will link to the issue tracker for implementation of the RFC)

# Summary
[summary]: #summary

Create a database benchmark for purposes of benchmarking different datastore.

# Motivation
[motivation]: #motivation

We are looking to evaluate alternative databases other than DynamoDB.  DynamoDB has limitations regarding querying that limits what we can do from a feature perspective.  However, it is extremely scalable and very fast.

Would like to have a repeatable benchmark that we can run to evaluate performace characteristics for recordset and recordset change data.  The benchmark can be used to tweak configuration parameters on the databases as well, in order to understand impact.

# Design and Goals
[design]: #design-and-goals

1. Should be configurable as to which repository implementations are being used.  For example, `MySQL` vs `DynamoDB`
1. Evaluate bulk insert performance times across different size bulk inserts.  The largest bulk insert is 10MM records
1. Evaluate read latency at estimated peak number of record sets (500MM record sets).  Read latency should be calculated at each of our "read" queries.
1. Wipe the and re-initialize the database before running against 

## Benchmark Design
[benchmark-design]: #benchmark-design
To evaluate read performance, the benchmark must load the database with ~500 million records across different zones.  We should have millions of zones loaded in the database, with different sizes.  The bulk of the zones can have a few record sets in them, as very large zones will be fewer in number.

1. 20 zones of 10,000,000 record sets (represents 200 million records)
1. 10 zones of 1,000,000 record sets (10 million records)
1. 10 zones of 100,000 record sets (1 million records)
1. 100 zones of 10,000 record sets (1 million records)
1. 1000 zones of 1,000 record sets (1 million records)
1. 10,000 zones of 100 record sets (1 million records)
1. 10,000,000 zones of 10 record sets (100 million records)

**Total Zones: ~10 million**
**Total Records: ~300 million**

The benchmark should

1. Insert both record sets _and_ record set changes.  This is done today when first loading a zone into VinylDNS.
1. Capture statistics for each zone loaded
1. Capture statistics for each stage loaded
1. Capture statistics for each query for 1 of each zone size loaded.  For example, `RecordSetRepository.listRecordSets` for 10 record sets, 100 record sets, 1000 record sets, etc. 7 in total.

## Benchmark Implementation
Since the VinylDNS repositories are all implemented in Scala, a java (or preferably Scala) based tool must be used to run the benchmarks.

Statistics for each step should be recorded in a manner that would allow it to be exported later.

# Drawbacks
[drawbacks]: #drawbacks

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

Need to workout the actual benchmark tool

# Outcome(s)
[outcome]: #outcome

Was this RFC implemented, subsumed by another RFC or implementation, in-waiting,
or discarded?

# References
[references]: #references

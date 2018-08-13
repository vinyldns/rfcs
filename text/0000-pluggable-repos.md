- Feature or Prototype: pluggable_repos
- Start Date: 2018-08-03
- RFC PR: #3
- Issue: 

# Summary
[summary]: #summary
The repositories are all `trait`s / interfaces that can be implemented in anyway.  We hard-code the implementation behind each repository's constructor.  For example, looking at the `UserRepository` we can see the pattern in the companion object...

```scala
object UserRepository {
  def apply(): UserRepository =
    DynamoDBUserRepository()
```

The goal will be to allow users to _configure_ what database implementation they are using so that they can use any database they want.  The database implementation will be dynamically loaded when VinylDNS starts up.

# Motivation
[motivation]: #motivation

VinylDNS is presently hard-coded to the choices made by Comcast to support our scale.  We run a combination of datastores:

* MySQL - used for zones and zone access.  Managing zone access required query patterns that were ill-suited for DynamoDB.  We also use MySQL for batch changes as well.
* DynamoDB - used for all other things, in particular Record data.  Due to the scale of Comcast, DynamoDB was a good choice as the number of records is in the 100s of millions.

Not all VinylDNS users (it could be argued very few) would want to run the same setup that we do.  To encourage adoption of VinylDNS, we need to allow users to "bring their own repositories" based on their databases of choice.  For example, even at very large scale it is possible to fit everything inside a large Postgres installation.

# Design and Goals
[design]: #design-and-goals

1. Support having any number of data stores for the repositories.  This allows us to mix-and-match different data stores per table, something currently done today with DynamoDB and MySQL
1. Support data store specific configuration. 


```yaml
data-stores = [
{
  type = "vinyldns.data.mysql"
  
  # Data store specific settings / connection settings go here
  url = ...
  user = ...
  password = ...

  # Repositories that use this data store are listed here
  user { // define user table name here }
  zone { // define zone table name here }
},
{
  type = "vinyldns.data.dynamodb"
  access-key = ...
  secret-key = ...
  endpoint = ...
  
  recordSet { // record set table properties go here, throughput, name }
  recordSetChange  { // record set change properties go in here}
}
]
```

# Drawbacks
[drawbacks]: #drawbacks

None.

# Alternatives
[alternatives]: #alternatives


# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?

# Outcome(s)
[outcome]: #outcome

Was this RFC implemented, subsumed by another RFC or implementation, in-waiting,
or discarded?

# References
[references]: #references

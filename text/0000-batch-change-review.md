_NOTE: This format comes from the
[Rust language RFC process](https://github.com/rust-lang/rfcs)._

- Feature or Prototype: Batch Change Review Process
- Start Date: 2019-04-03
- RFC PR: (leave this empty, populate once the PR is created with a github link to the PR)
- Issue: (once approved, this will link to the issue tracker for implementation of the RFC)

# Summary
[summary]: #summary

Implement a review step in the batch change process between validation and processing.

# Motivation
[motivation]: #motivation

Example 1: If a zone does not exist in Vinyl we may want to add that zone to Vinyl and then continue processing the batch change.
Example 2: Certain record types may need manual review and approval.

# Design and Goals
[design]: #design-and-goals

User submits a batch change (in the portal):
 1. hits submit
 1. run validations
 1. tell user everything is okay
 1. user hits okay to move batch change into processing or hits cancel and continues modifying batch change
 
 exceptions:
  - validations hard fail
  - validation warnings
 
DNS technician reviews pending requests:
  1. technician checks requests
  1. technician reviews
  1. techician clicks Approve (or deny)
  1. batch change is submitted for processing (or marked as Denied)
  
# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- What validations/rules require manual intervention?

What parts of the design are still TBD?

# Outcome(s)
[outcome]: #outcome

Was this RFC implemented, subsumed by another RFC or implementation, in-waiting,
or discarded?

# References
[references]: #references

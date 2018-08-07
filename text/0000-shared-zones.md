_NOTE: This format comes from the
[Rust language RFC process](https://github.com/rust-lang/rfcs)._

- Feature or Prototype: (fill me in with a unique identity, e.g.
  my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty, populate once the PR is created with a github link to the PR)
- Issue: (once approved, this will link to the issue tracker for implementation of the RFC)

# Summary
[summary]: #summary

Shared zones allow for the management of certain zones that would otherwise be very difficult to manage access via ACL rules.  We introduce an alternative access control model for these kinds of zones.

# Motivation
[motivation]: #motivation

Zones such as IP4 and IP6 reverse zones, and large "common" zones are used by many different users and groups.  DNS records in these zones change frequently.  Further, they have no easily identifable "owner".  For example, contiguous IP space maybe granted to an NDC team; however, they typically do not govern (nor do they want to govern), who creates PTR records to claim specific addresses in that IP space.

Shared zones will allow these kinds of zones (unowned / shared by many) to use an alternative access control model.  This allows users to freely and safely make changes in these zones; but still prevents stomping on DNS records.

# Design and Goals
[design]: #design-and-goals

## Shared Zone Overview
When a zone is setup, the zone can be specified as "shared".  This means that any VinylDNS user can potentially make any change to the shared zone.  The following define the features and functions of shared zones

* Every record set in a shared zone will have an associated (optional) `owner`.  The owner will be the VinylDNS group that created the record set.
* For shared zones that are imported into VinylDNS, all record set owners will initially be unassigned.  This implies that anyone can potentially operate on these record sets.
* Any VinylDNS user can create a new record set in any shared zone, assuming that the record set is in good order.
* For existing record sets, updates and deletes can only be made by users who are a member of the record owning group.
If the owning group is unassigned, the change can be applied to the record set.

In evaluating access controls, the current access models will supercede record set ownership.

1. VinylDNS Admins can make any change
1. Zone admins can change all records in a zone
1. ACL rules can grant permissions to change records
1. Finally, record ownership can allow access to record modification if the user is not authorized on the previous

## Record Ownership Overview
Record ownership is tightly coupled to Shared Zones (shared zones could not be implemented without record ownership); therefore, we include them here.

A "record owner" is the group that is designated as the "owner" of a particular record set.  A new _optional_ parameter will be passed in on all record set operations to indicate the "group" that the change is made "on behalf of".  There are multiple ways that the owner is assigned:

1. If a record set is imported in, initially the owner is unassigned.  Updates to the record set would assign the owner if the `owner` parameter is defined on the request.  If the `owner` is not defined, then it will remain unchanged.  
1. If a record set does not yet exist, and the `owner` parameter is defined, the `owner` group will be assigned to the new record set when it is successfully created.
1. VinylDNS, zone admins, and users that are members of the `owner` group can modify record ownership at any time.  This includes _reassigning_ and _unassigning_ record ownership for a particular record set.



# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?

# Outcome(s)
[outcome]: #outcome

Was this RFC implemented, subsumed by another RFC or implementation, in-waiting,
or discarded?

# References
[references]: #references

_NOTE: This format comes from the
[Rust language RFC process](https://github.com/rust-lang/rfcs)._

- Feature or Prototype: shared_zones
- Start Date: 2018-08-07
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

## Access Control Checks
In evaluating access controls, the current access models will supercede record set ownership.

1. VinylDNS Admins can make any change
1. Zone admins can change all records in a zone
1. ACL rules can grant permissions to change records
1. Finally, record ownership can allow access to record modification if the user is not authorized on the previous checks.  Record ownership is only consulted IFF shared zones are enabled on the zone the record lives in.

## Record Ownership Overview
Record ownership is tightly coupled to Shared Zones (shared zones could not be implemented without record ownership); therefore, we include record ownership features in this RFC.

A "record owner" is the group that is designated as the "owner" of a particular record set.  A new _optional_ parameter will be passed in on all record set operations to indicate the "group" that the change is made "on behalf of".  There are multiple ways that the owner is assigned:

1. If a record set is imported in, initially the owner is unassigned.  Updates to the record set would assign the owner if the `owner` parameter is defined on the request.  If the `owner` is not defined, then it will remain unchanged.  
1. If a record set does not yet exist, and the `owner` parameter is defined, the `owner` group will be assigned to the new record set when it is successfully created.
1. VinylDNS, zone admins, and users that are members of the `owner` group can modify record ownership at any time.  This includes _reassigning_ and _unassigning_ record ownership for a particular record set.

## Shared Zone Design
Zones will be designated as `shared` by setting a `shared` flag (boolean) on the zone itself.  The flag should default to _false_.  *Note: This flag is actually already present, so the only change will be to allow the flag to be set from the API and UI.*  The `shared` flag can only be set:

1. Add a `shared` flag to the `JSON` Zone model that will allow the alternative access control model.
1. Add a `shared` flag in the portal when connecting to a zone (defaults to off)
1. Add a `shared` flag in the portal's Zone Management screen that allows zone admins and VinylDNS admins to turn on/off shared zone management.  Turning the flag off should simply disable the shared zone access check for future updates, and should _not_ attempt to clean up or clear the owners on the record sets.
1. Add a `shared` query parameter when listing and searching zones, should default to `false` if not present.
1. Add a `shared` check-box in the portal next to the search box that allows the users to navigate and search through shared zones.
1. Add a `shared` column in the `zone` table (default false)

## Record Ownership Design
Record ownership will be _optional_ in the system.  The property will be added to all `RecordSet`s, with the possibility of being `null`.

1. Modify the `RecordSet` JSON to have an `ownerGroupId` property.  The `ownerGroupId` is to be a VinylDNS `groupId` that the calling user is a member of.  
1. For `Update RecordSet Requests`; the owner can be _reassigned_ by changing the `ownerGroupId` to another group that the caller is a member of.
1. For `Update RecordSet Requests`; the owner can be _unassigned_ by _clearing_ the `ownerGroupId` in the JSON request sent.
1. In the portal's Zone page, allow users to designate a record owner when creating or updating a record set.  
1. Users should _not_ be allowed to specify an `ownerGroupId` if the zone is not shared.  If the `ownerGroupId` is provided and the zone is not shared, the `ownerGroupId` will be discarded.
1. In the portal's Batch Change page, allow users to designate a record owner for the entire batch change.  A drop-down list of groups that the user is a member of should be provided to allow the user to choose _one_ group.
1. For Batch Changes, do not reject changes if the `ownerGroupId` is specified and the zone is not shared.  As batch changes can span shared and un-shared zones, we cannot enforce that requirement across all changes in the batch.

## Example Setup
This example is to illustrate use cases around setting up and using shared zones.

**Setting up a shared zone via the API**
API request is made `POST /zones` with the following

```json
{
  "adminGroupId": "9b22b686-54bc-47fb-a8f8-cdc48e6d04ae",
  "name": "dummy.",
  "email": "test@example.com",
  "shared": true
}
```

The zone is synced in, all existing record sets are loaded into VinylDNS with an `ownerGroupId = null`.

**Creating a new "owned" record set in a shared zone via the API**
API request is made `POST /zones/<id>/recordsets` to a zone that is shared with the following

```json
{
    "type": "A",
    "zoneId": "<id>",
    "name": "foo",
    "ttl": 38400,
    "records": [
        {
            "address": "1.1.1.1"
        }
    ],
    "id": "8306cce4-e16a-4579-9b19-4af46dc75853",
    "ownerGroupId": "b34f8d18-646f-4843-a80a-7c0d58a22bf5"
}
```

Processing:
1. The user making the request must be a member of the group specified by `ownerGroupId` (or an admin)
1. If the user making the request is not a member of the group specified, a `403` error will be returned
1. If the zone is not shared, the `ownerGroupId` will be discarded.
1. If the zone is shared, the `ownerGroupId` will be saved on the record set

**Updating ownership for an "owned" record set in a shared zone via the API**

`PUT /zones/<id>/recordsets/<id>`

```json
{
    "type": "A",
    "zoneId": "<id>",
    "name": "foo",
    "ttl": 38400,
    "records": [
        {
            "address": "1.1.1.1"
        }
    ],
    "id": "8306cce4-e16a-4579-9b19-4af46dc75853",
    "ownerGroupId": "newGroupId"
}
```

Processing:
1. The user making the request must be a member of the _new_ owner group (or an admin)
1. The user making the request must be a member of the _previous_ owner group (or an admin)
1. If the user making the request is not a member of either group (or an admin), a `403` error will be returned
1. The record set is updated, the `ownerGroupId` is set to the value provided

**Deleting an "owned" record set in a shared zone via the API**

`DELETE /zones/<id>/recordsetes<id>`

Processing:
1. The user making the request must be a member of the owner group for the record set (or an admin)
1. If the user is not a member of the `ownerGroupId` of the record set being deleted, the request will be rejected with a `403` error
1. If the user is a member, the record set will be deleted normally
1. The "owner group" is not preserved after deletion of a record set.  The record set is assumed to be "unassigned" on deletion.

**Submitting a Batch Change**
When submitting a batch change, a new parameter `ownerGroupId` is added to the JSON as shown in the example.  The `ownerGroupId` is optional (as not all users manage records in shared zones).

`POST /zones/batchrecordchanges`

```
{
    "comments": "this is optional",
    "ownerGroupId": "12345",
    "changes": [
        {
            "inputName": "example.com.",
            "changeType": "Add",
            "type": "A",  
            "ttl": 3600, 
            "record": {
                "address": "1.1.1.1"
            } 
        }, 
        {
            "inputName": "192.0.2.195",
            "changeType": "Add",
            "type": "PTR", 
            "ttl": 3600,
            "record": {
                "ptrdname": "ptrdata.data."
            }
        }
    ]
}
```

Processing

1. Ensure that the user is a member of the `ownerGroupId` if provided
1. If the user is not a member of the `ownerGroupId` provided, return a `403` error
1. For each change that is generated, if the zone is shared and the `ownerGroupId` is provided, consult record ownership.
1. If the change is disallowed, the entire batch change should be rejected.  The appropriate error should be returned on any change(s) that failed indicating that the user is not authorized because the record ownership check failed.


# Drawbacks
[drawbacks]: #drawbacks

This does loosen up access control rules.  The record ownership checks and ownership reassignment do "lock down" things in a sense that we do not have an environment where users can stomp over other users records.

# Alternatives
[alternatives]: #alternatives

1. As opposed to "open" access to _all_ records in the zone, we could further restrict which _records_ were "open" to the public using a new ACL rule.  For example, we could say "A, AAAA, CNAME" records are "shared".  "Shared" access indicates that access controls will follow what has beend defined in this document.  The downside is this complicates the ACL rules further, but makes shared access extremely flexible.

# Unresolved questions
[unresolved]: #unresolved-questions

## Impact on what a user sees
For users that do not "own" zones and _exclusively_ work in shared zones, what will they see in the "zones" list?  This is particularly problemsome as users will see "all" shared zones, but only care about the few that they actually have made updates to.

One solution to this problem is that we could idempotently insert a record in the "zone access" table.  The zone access table governs access for users and groups to zones.  It is how the system generates "my zones", limiting the list of zones that you have access to.  If a group is assigned an owner on any record in a zone, we create a zone access entry for that group to that zone.

# Outcome(s)
[outcome]: #outcome

Was this RFC implemented, subsumed by another RFC or implementation, in-waiting,
or discarded?

# References
[references]: #references

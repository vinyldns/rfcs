_NOTE: This format comes from the
[Rust language RFC process](https://github.com/rust-lang/rfcs)._

- Feature or Prototype: ptr_correspondance
- Start Date: 2019-01-03
- RFC PR: TBD
- Issue: TBD

# Summary
[summary]: #summary

PTR records facilitate reverse lookups via utilities like `whois` in order to determine the domain name for a given IP address.  In general, it is best practice to ensure that every A and AAAA record has a exactly **one matching** PTR record in a reverse zone.  Similarly, each PTR record set should have exactly one record in it that corresponds to the A or AAAA record that it points to.

Due to the nature of DNS, these records exist in _separate_ DNS zones.  In order to facilitate this best practice, users have to 1) be aware of the convention and 2) ensure that they are making updates in both the correct forward and reverse zone.

PTR Correspondance will make it simpler to enable this best practice for VinylDNS users by giving them the option to keep forward and reverse records in sync.

# Motivation
[motivation]: #motivation

Ideally, every forward record points to exactly one reverse record, and that reverse record points to exactly one forward record.  This best practice is cumbersome in VinylDNS today, as users have to make 2 separate transactions in order to keep the forward and reverse records aligned.  The present process follows:

1. User creates an A record `foo` in the `bar.com` zone: `foo.bar.com A 300 192.168.0.1`
2. User creates a PTR record `1` in the `0.168.192.in-addr.arpa` zone: `1 PTR 300 foo.bar.com`

Similarly, when a user goes to _update_ the record, they need to make 3 separate transactions:

1. User updates `foo` in the `bar.com` zone to a new IP address: `foo.bar.com A 300 10.121.0.200`
2. User deletes the PTR record `1` in the `0.168.192.in-addr.arpa` zone: `1 PTR 300 foo.bar.com`
3. User creates the PTR record `200` in the `0.121.10.in-addr.arpa` zone: `200 PTR 300 foo.bar.com`

Finally, when a user _deletes_ the record, they need to make sure to delete both the forward and reverse records in 2 transactions.

PTR correspondance attempts to make this simple for the user, facilitating the correspondance between A, AAAA and their PTR records.

# Design and Goals
[design]: #design-and-goals

The application has 2 kinds of users:

1. **Casual users** - these are users who simply want to give names to things.  They generally are not aware of the complexity of DNS, or even what a DNS record is.
2. **Advanced users** - these are users who are generally well-versed in DNS, and will be familiar with the concepts of PTR, A, AAAA records and the idea of PTR correspondance.

The recommended approach is to facilitate PTR correspondance via an option when users manage records.  We do not want to manage the correspondance for users automatically, as there are valid use cases when correspondance is not needed as well as times when a PTR record _should_ have multiple forward records associated with it.

Casual users generally use the _batch change_ screen to make their changes; whereas advanced users will typically manage their zones via the _zone_ screen.

## Batch Screen

1. For any ADD record change for an A or AAAA record, allow the user to check a box to automatically update the corresponding PTR record.  This check box should be **checked** by default.
1. For any DELETE record change for an A or AAAA record, allow the user to check a box to automatically delete the corresponding PTR record.  This check box should be **checked** by default.
1. For any ADD record change for a PTR record, allow the user to check a box to automatically update the corresponding A or AAAA record.  This check box should be **checked** by default.
1. For any DELETE record change for a PTR record, allow the user to check a box to automatically delete the corresponding A or AAAA record.  This check box should be **checked** by default.

## Zone Screen

1. When a user chooses to Create a new A or AAAA record set, allow the user to check a box to automatically update the corresponding PTR records.  **checked** by default.
1. When a user chooses to Update an A or AAAA record set, allow the user to check a box to automatically update the corresponding PTR records.  **checked** by default.
1. When a user chooses to Delete an A or AAAA record set, allow the user to check a box to automatically delete the corresponding PTR records.  **checked** by default.
1. When a user chooses to Create a new PTR record set, allow the user to check a box to automatically update the corresponding A or AAAA records.  **checked** by default.
1. When a user chooses to Update a PTR record set, allow the user to check a box to automatically update the corresponding A or AAAA records.  **checked** by default.
1. When a user chooses to Delete a PTR record set, allow the user to check a box to automatically delete the corresponding A or AAAA records.  **checked** by default.


## Use Cases

### Create an A or a AAAA record set with PTR correspondance enabled

_Note: This can be entered via batch or zone screen, the net action is to create an A record set_

1. System detects that a new A record set is to be created and the PTR correspondance flag is set to `true`
2. For each IP address in the A record set, lookup the corresponding PTR record in our database to see if it already exists
3. If it exists in VinylDNS, ensure that the user has access to make the change of the PTR record
4. Create or update the PTR record for each record in the A record set.  The `ptrdname` should be set to the FQDN of the a record

        
#### Exception Case: PTR Record does not exist in VinylDNS database

1. If it does not exist, look up the corresponding PTR record in the backend DNS server
    1. If the record does not exist in the backend, then proceed with creating the PTR record normally
    2. If the record exists in the backend but not in VinylDNS, first load the record set into VinylDNS with the record data set to what is in the DNS backend (effectively syncing the record set), then issue an Update record set change normally.  This would result in 2 separate changes, one to load the record set and a second to update it
    
**Note: The exception condition is new and should be discussed.  The motivation for making this change is because we are doing changes behind the scenes on behalf of the user to maintain correspondance.  The alternative would be to reject the change as we do not have the record set (which follows what we do today), but the fear is that most users would simply "uncheck" the checkbox to move forward.  Doing that would be counter to this entire feature!  I believe it is incumbent upon VinylDNS to ensure correspondance as much as possible.**
    
#### Exception Case: The user does not have access to the PTR record

1. If the user does not have access to update the PTR, reject the request (batch change request or create record set request).  Provide information in the response as to who has ownership to help possibly gain the right access.

#### Exception Case: Different PTR records are found pointing to the new A or AAAA record set

It is possible that other PTR record sets are found in VinylDNS that point to the new A or AAAA record set.  _Note: it would not be possible to query the DNS backend for this kind of information; however, it is possible to do the lookup for PTR records matching the FQDN of the forward record._

1. For each PTR record found, attempt to delete the record set.  If the user does not have permission, reject the change.

### Create a PTR record set with PTR correspondance enabled

_Note: This can be entered via batch or zone screen, the net action is to create an A record set.  This case should mirror the A or AAAA record set case_

1. System detects that a new PTR record set is to be created and the PTR correspondance flag is set to `true`
2. Check that there is only 1 record in the PTR record set
3. Lookup the corresponding A or AAAA record in our database to see if it already exists
4. If it exists in VinylDNS, ensure that the user has access to make the change of the A or AAAA record
5. Create or update the A or AAAA record set

#### Exception Case: Multi-record PTR with PTR correspondance enabled

1.  By the definition of PTR correspondance, there can be exactly **one** record in the record set.  Reject the request with a failure message indicating that the request is malformed
        
#### Exception Case: A or AAAA Record does not exist in VinylDNS database

1. If it does not exist, look up the corresponding A or AAAA record in the backend DNS server
    1. If the record does not exist in the backend, then proceed with creating the A or AAAA record normally
    2. If the record exists in the backend but not in VinylDNS, first load the record set into VinylDNS with the record data set to what is in the DNS backend (effectively syncing the record set), then issue an Update record set change normally.  This would result in 2 separate changes, one to load the record set and a second to update it
    
#### Exception Case: The user does not have access to the A or AAAA record

1. If the user does not have access to update the A or AAAA record, reject the request (batch change request or create record set request).  Provide information in the response as to who has ownership to help possibly gain the right access.

### Update an A or AAAA record set with PTR correspondance enabled

_Note: This can be entered via batch or zone screen, the net action is to create an A record set_

1. System detects an update to an existing A or AAAA record set with PTR correspondance enabled
2. For each IP address in the A record set, lookup the corresponding PTR record in our database to see if it already exists
3. If it exists in VinylDNS, ensure that the user has access to make the change of the PTR record
    1. If an address is _removed_ from the record set as part of the update, issue a **Delete** if and only if there is only a single record in the record set.  If there are multiple records in the PTR record set, **reject** the change
    2. If an address is _added_ to the record set as part of the update, issue an **Update** to the PTR record set if and only if there is a single record in the record set.  If there are multiple records in the PTR record set, **reject** the change
    
### Update a PTR record set with PTR correspondance enabled

_Note: This can be entered via batch or zone screen, the net action is to create an A record set_

1. System detects an update to an existing PTR record set with PTR correspondance enabled
2. If there is more than one record in the record set, reject the change
3. Lookup the corresponding A or AAAA record in our database to see if it already exists
4. If it exists in VinylDNS, ensure that the user has access to make the change of the PTR record
    1. If an address is _removed_ from the record set as part of the update, issue a **Delete** if and only if there is only a single record in the record set.  If there are multiple records in the PTR record set, **reject** the change
    2. If an address is _added_ to the record set as part of the update, issue an **Update** to the PTR record set if and only if there is a single record in the record set.  If there are multiple records in the PTR record set, **reject** the change


# Drawbacks
[drawbacks]: #drawbacks


# Alternatives
[alternatives]: #alternatives


# Unresolved questions
[unresolved]: #unresolved-questions


# Outcome(s)
[outcome]: #outcome



# References
[references]: #references

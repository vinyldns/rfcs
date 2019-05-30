_NOTE: This format comes from the
[Rust language RFC process](https://github.com/rust-lang/rfcs)._

- Feature or Prototype: Batch Change Review Process
- Start Date: 2019-04-03
- RFC PR: (leave this empty, populate once the PR is created with a Github link to the PR)
- Issue: https://github.com/vinyldns/vinyldns/issues/659

# Summary
[summary]: #summary

Implement a review step in the batch change process between validation and processing.

# Motivation
[motivation]: #motivation

Example 1: If a zone does not exist in VinylDNS we may want to add that zone to VinylDNS and then continue processing the batch change.

Example 2: Certain record types may need manual review and approval.

# Design and Goals
[design]: #design-and-goals

- Maintain existing batch change flow if the batch change passes all validations. No manual intervention required.

- If the batch change fails validation for one or more of the permissible errors, do not send for processing, instead designate as requiring manual intervention.

DNS administrator reviews pending requests:
  1. DNS administrator checks list of pending requests
  1. DNS administrator reviews a pending request
  1. DNS administrator Approves (or Rejects) the request
  1. An e-mail is sent to the batch change submitter to let them know the outcome of the review
  1. If approved, batch change is submitted for processing, resuming current batch change workflow

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

1. `BatchChange` would be tightly coupled with the concept of pending/submitted state.
1. Support for new request types such as zone creation and out-of-band requests would require changing a lot of how current `BatchChange` flow works.
1. A lot more fields would be added to `BatchChange` which are only required for pending state.

# Alternatives
[alternatives]: #alternatives
<details>
<summary>Implement batch change request as an optional pre-approval process to batch change.</summary>

## Design and Goals
1. User submits `BatchChangeRequest` (either through portal form or API request)
1. Validations are run on `BatchChangeRequest` to separate requests that need to be processed manually out-of-band and those that will break off into batch change validations
  1. Most existing requests can flow through batch change validations (eg. create/update/delete supported record types); need to add a create zone and custom request which will be flagged for out-of-band processing. This is also how users would have to submit requests that would otherwise get rejected through normal batch change processing (eg. creating/modifying a high-value domain or unsupported name server, etc.)
1. If there are any batch change validations that fail, reject `BatchChangeRequest` outright (`Denied` status)
1. If all validations pass:
  1. If there are _no_ out-of-band changes, VinylDNS can auto-submit as a batch change for processing, possibly controlled by a flag on the request to auto-complete if possible. (`Implemented` status) *NOTE*: Could have all good batch change requests move into a pending state that needs to be manually approved. (`Submitted` status)
  1. If there are any out-of-band changes in addition to normal batch changes, each change will have a `ManualBatchChangeRequest` item created. (`Submitted` status) Each of these items will have to be flagged as completed before the other in-band `BatchChangeRequest` can be converted to a `BatchChange`.
  1. If there are _only_ out-of-band changes, completing all `ManualBatchChangeRequest` items and submitting will simply mark the `BatchChangeRequest` as `Implemented`.
1. User can cancel a `BatchChangeRequest` that is in `Submitted` status (`Canceled` status)
1. A technician can approve or reject the `BatchChangeRequest`, with the stipulation that approving requires any existing `ManualBatchChangeRequest`s to be marked completed. Approval is a _final_ process which will spawn a corresponding `BatchChange`, if appropriate. This action cannot be undone.

### Pros
1. `BatchChange` data will not be contaminated by tightly coupling a pending batch state, which _may_ consist of solely out-of-band items.
1. New tables and data structures would be created for `BatchChangeRequest` and `ManualBatchChangeRequest`, which allows the data to be cleaner.
1. A lot of the ground work from `BatchChange` can be reused.
1. Could allow a user to edit a `BatchChangeRequest` that is in `Submitted` status.
1. Current implementation of `BatchChange` flow would be completely untouched.
1. Don't need to add new optional attributes to `BatchChange` that would only be relevant for pending support.

### Cons
1. More coding changes required, though a lot more work can happen simultaneously.

### Considerations
1. Since `BatchChangeRequest` would be validating the changes that would be going through `BatchChange`, it may be a good idea to add an overload for `BatchChangeService` which skips validations for `BatchChangeRequest`s that succeed validations to avoid "double processing".

### Design notes
1. Out-of-band items as of this time would only be zone create and some type of custom request. The custom request should simply be a a freeform field for the submitter's comment and another freeform field for DNS technician comments.
</details>

# Unresolved questions
[unresolved]: #unresolved-questions

- What validations/rules require manual intervention?

# Outcome(s)
[outcome]: #outcome

Was this RFC implemented, subsumed by another RFC or implementation, in-waiting,
or discarded?

# References
[references]: #references
